# **Server-Side Conversion Tracking: состояние индустрии и архитектурный разбор (2026)**

## **1\. Краткое резюме (TL;DR)**

* **Ментальная модель:** Серверный трекинг конверсий (S2S / CAPI) — это не замена браузерным пикселям, а **гибридная система доставки сигналов**. Браузер ловит контекст клика и передает события быстро, но теряет 20–40% данных; сервер гарантирует доставку бизнес-критичных фактов (регистрация, депозит, чарджбэк) с привязкой к первому касанию.  
* **Масштаб vs корректность:** 50k регистраций и 5k депозитов в сутки — это **менее 1 события в секунду в среднем и \~3.2 события/с на 5× пике**. Задача решается не горизонтальным масштабированием, а **строгой идемпотентностью, управлением состояниями доставки и сверкой (reconciliation)**.  
* **Ключевые архитектурные решения:**  
  1. **Минимальный след в монолите:** PHP-монолит не знает про рекламные сети; он принимает один непрозрачный токен attribution\_token при регистрации и пишет бизнес-события в таблицу **Transactional Outbox** в MariaDB.  
  2. **Единый Attribution Store:** Отдельный легкий сервис (или модуль) сохраняет связку attribution\_token → все Click ID (gclid, fbclid, click\_id и др.), UTM и consent-флаги на первом хите.  
  3. **Изолированные адаптеры доставки:** Kafka-консьюмеры \+ БД-ориентированный **Delivery Ledger** (журнал доставок) со стейт-машиной и персональными лимитами/ретраями под каждую сеть.  
  4. **Сеть B (таймауты без дедупликации):** Строгое правило **«никогда не ретраить статус UNKNOWN вслепую»**; событие уходит в карантин до сверки с ежедневным отчетом.  
  5. **Сеть C (аффилиаты, деньги, чарджбэки):** Финансовый **Payout Ledger** с ключом (partner\_id, click\_id, event\_type), длинные ретраи (до 72+ часов) и автоматическая ежемесячная сверка.  
* **Главные риски:** (1) Двойной учет конверсий при рассинхронизации event\_id между GTM и сервером; (2) передача данных без согласия пользователя (нарушение GDPR/DMA и бан рекламных аккаунтов); (3) потеря атрибуции при переходах между зеркалами домена; (4) финансовые потери при несвоевременной отправке отмены (chargeback) аффилиатам.

## **2\. Как всё устроено: базовая картина**

### **2.1. Жизненный цикл атрибуции клика (End-to-End)**

Атрибуция (attribution) — это процесс связывания целевого действия пользователя (конверсии) с конкретным рекламным кликом или показом, который привел этого пользователя.

Фрагмент кода

```
sequenceDiagram
    participant AdNet as Рекламная / Партнерская сеть
    participant Browser as Браузер пользователя
    participant Landing as Лендинг / Фронтенд
    participant AttrSvc as Attribution Service
    participant Core as PHP-монолит (MariaDB)
    participant Delivery as Conversion Delivery Service

    AdNet->>Browser: 1. Показ рекламы и клик (редирект с ?click_id=XYZ)
    Browser->>Landing: 2. Загрузка страницы с Click ID в URL
    Landing->>AttrSvc: 3. Сохранение Click ID + User-Agent + IP + Consent
    AttrSvc-->>Landing: 4. Выдача непрозрачного attribution_token (1st-party cookie)
    Browser->>Core: 5. Регистрация (передача attribution_token) + Депозит
    Core->>Core: 6. Транзакция: запись user/deposit + Outbox Event
    Core-->>Delivery: 7. Событие попадает в Kafka
    Delivery->>AttrSvc: 8. Обогащение: получение Click ID и хеша email
    Delivery->>AdNet: 9. Отправка S2S конверсии (REST / GET postback)
```

**Реальные параметры кликов и их семантика (факт, документация платформ):**

| Платформа | Параметры URL / Cookies | Назначение и особенности |
| :---- | :---- | :---- |
| **Google Ads** | gclid, gbraid, wbraid | gclid — основной ID клика. wbraid (web-to-app) и gbraid (app-to-web) введены для iOS/Safari, где gclid часто урезается или не поддерживается из\-за ограничений приватности. |
| **Meta (Facebook/IG)** | fbclid (URL), \_fbc, \_fbp (cookies) | fbclid приходит в URL. Пиксель или сервер формирует cookie \_fbc в формате fb.1.\<timestamp\_ms\>.\<fbclid\>. Cookie \_fbp — идентификатор браузера (fb.1.\<timestamp\_ms\>.\<random\>). |
| **TikTok** | ttclid (URL), \_ttp (cookie) | ttclid — уникальный ID клика. \_ttp — first-party cookie для улучшения сопоставления (matching). |
| **Microsoft Ads** | msclkid | Аналог gclid для экосистемы Bing/Microsoft Advertising. |
| **Аффилиат-сети (CPA)** | click\_id, clickid, sub\_id, sub1..sub5, aff\_id | Произвольные параметры трекера партнера. В URL обычно передается уникальный ID клика, который необходимо вернуть обратно в S2S postback без изменений. |

### **2.2. Как работает пиксель и почему он теряет 20–40% событий**

**Браузерный пиксель (browser tag)** — это фрагмент JavaScript (часто загружаемый через Google Tag Manager), который:

1. Читает cookies и параметры URL.  
2. Слушает события DOM или dataLayer.  
3. Отправляет HTTP-запрос (fetch, navigator.sendBeacon или невидимый 1x1 image request) на домен рекламной сети.

**Почему потери составляют 20–40% (реалистичные диапазоны, индустриальные данные 2024–2026):**

* **Блокировщики рекламы (Ad Blockers):** uBlock Origin, AdGuard, встроенные блокировщики Opera/Brave блокируют домены трекеров (connect.facebook.net, googletagmanager.com, google-analytics.com). В зависимости от географии и аудитории (особенно десктоп и tech-аудитория) потери составляют **15–30%** (индустриальная практика; отчеты PageFair/Blockthrough и вендоров sGTM).  
* **Safari ITP (Intelligent Tracking Prevention) и Link Tracking Protection (LTP):**  
  * В Safari (iOS и macOS) cookies, установленные через JavaScript (document.cookie), живут **максимум 7 дней** (а если переход был с домена, классифицированного как трекер, с параметром в URL — **всего 24 часа**).  
  * Начиная с iOS 17 / macOS Sonoma (2023) и далее в 2024–2026, **Link Tracking Protection** в Private Browsing (а также в Mail и Messages) **вырезает известные параметры трекинга** (fbclid, gclid, msclkid, ttclid) прямо из URL до загрузки страницы (факт, документация WebKit).  
* **Firefox ETP (Enhanced Tracking Protection) и Brave:** В режиме Strict (Firefox) и по умолчанию (Brave) режут трекинговые скрипты и очищают query-параметры (Query Parameter Stripping).  
* **Статус Chrome и Third-Party Cookies (3PC) в 2025–2026:** В отличие от ранних планов Google полностью отключить 3PC, в **апреле 2025 года** Google официально объявила, что **не будет принудительно отключать third-party cookies в Chrome**, оставив существующую модель выбора пользователя в настройках, и свернула часть инициатив Privacy Sandbox (факт, блог Privacy Sandbox, 2025). Однако тренд на ограничения на уровне iOS/Safari, блокировщиков и регуляторов (GDPR/DMA) сохранил долю потерь клиентских событий на уровне **20–40%**.  
* **In-App браузеры и короткие сессии:** Пользователи из Instagram/TikTok/Telegram открывают сайт во встроенном WebView, где cookies изолированы от основного браузера. При переходе на оплату или открытии ссылки в Safari контекст теряется.  
* **Зеркала доменов (Mirror Domains):** Если пользователь кликнул на site-mirror1.com, а зарегистрировался или пополнил счет на site-mirror2.com, first-party cookies и localStorage первого домена недоступны на втором (Same-Origin Policy).

### **2.3. Client-side vs Server-side: почему стандарт — гибрид**

| Критерий | Client-side (Пиксель / GTM) | Server-side (S2S / CAPI) |
| :---- | :---- | :---- |
| **Видимость контекста браузера** | Высокая: cookies, User-Agent, экран, реферер, просмотры страниц, клики по кнопкам. | Низкая: сервер видит только то, что явно передал фронтенд или сохранил бэкенд. |
| **Устойчивость к блокировщикам** | Низкая (блокируется на уровне браузера/DNS). | Высокая (запрос идет с вашего сервера напрямую в API сети). |
| **Достоверность бизнес-фактов** | Низкая (пользователь может закрыть вкладку до загрузки «Thank you page»; данные легко подделать). | Абсолютная (событие генерируется после коммита в БД / подтверждения платежного шлюза). |
| **Задержка и контроль** | Мгновенная отправка, но нет гарантий доставки и ретраев. | Гарантированная доставка (очереди, ретраи), возможность отправлять оффлайн-события и чарджбэки. |

**Почему индустрия использует гибридную схему (факт, рекомендации Meta и Google):**

Пиксель обеспечивает богатые поведенческие сигналы (верх воронки: PageView, ViewContent) и мгновенную привязку к сессии браузера. Сервер гарантирует доставку нижних, денежных этапов воронки (Registration, FirstDeposit) и защищает от потерь. Чтобы конверсия не засчиталась дважды, **оба канала отправляют одинаковый event\_id**, а рекламная платформа выполняет дедупликацию.

### **2.4. Как сети используют сигналы: ML-аукционы vs аффилиат-учет**

Принципиально важно разделять два класса получателей:

1. **Рекламные сети (Meta, Google, TikTok — аукцион \+ ML):**  
   * Используют конверсии для обучения алгоритмов ставок (**Value-Based Bidding**, tCPA, tROAS).  
   * **Learning Phase (фаза обучения):** Для стабилизации алгоритма Meta/TikTok обычно требуется \~50 конверсий на группу объявлений в неделю. Потеря 30% событий не дает кампании выйти из фазы обучения, стоимость привлечения (CPA) растет.  
   * **Event Match Quality (EMQ):** Оценка качества сопоставления (у Meta от 1 до 10\) — насколько полно переданы идентификаторы (Click ID, хеши email/телефона, IP, User-Agent). Чем выше EMQ, тем точнее атрибуция.  
   * **Вред дублей и опозданий:** Дубликат конверсии завышает расчетный ROAS, заставляя алгоритм агрессивно переплачивать за некачественный трафик. Событие, пришедшее через неделю, искажает атрибуцию и не помогает в реальном времени.  
2. **Аффилиат-сети / CPA-партнеры (финансовый учет):**  
   * Здесь нет ML-аукциона. Каждый подтвержденный S2S postback — это **строка в акте сверки и прямое начисление денег вебмастеру** (например, \$100 за первый депозит).  
   * Дубль \= двойная выплата (прямой убыток компании).  
   * Потеря события \= недоплата партнеру, конфликт и отключение трафика.  
   * Чарджбэк (отмена платежа) без отправки отмены партнеру \= оплата фрода.

## **3\. Состояние рынка и API в 2026**

### **3.1. Сравнительная таблица рекламных API**

**Примечание по актуальности:** Данные отражают состояние официальной документации платформ на 2025–2026 годы. Параметры, которые могут меняться в зависимости от уровня аккаунта (tier) или точной версии API, помечены \[UNVERIFIED — проверить в личном кабинете\].

| Платформа | Стиль API и Auth | Batch / Rate Limits | Лимит возраста события | Дедупликация и окно | Правила хеширования PII (SHA-256) |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **Meta Conversions API (CAPI)** | REST (Graph API, JSON). Auth: System User Access Token. | До **1 000** событий в батче. Rate limit динамический (зависит от аккаунта/приложения, стандартно сотни req/min). | **7 дней** с момента события (event\_time). Старше — отклоняются. | По паре event\_name \+ event\_id (плюс external\_id/fbp). Окно дедупликации — **48 часов**. | Lowercase, trim пробелов. Телефон: только цифры с кодом страны (E.164 без \+). Email: trim \+ lowercase. |
| **Google Ads (Enhanced Conversions / Data Manager API)** | gRPC / REST (Google Ads API v17–v20+ / Data Manager API). Auth: OAuth 2.0 \+ Developer Token. | До **2 000** конверсий на запрос (UploadClickConversions). Rate limits по квотам Google Ads API (Basic/Standard access). | До **90 дней** (в пределах окна клика gclid, обычно 90 дней). Рекомендуется отправлять в течение 24 часов. | По order\_id (transaction\_id) \+ conversion\_action \+ conversion\_date\_time. | Lowercase, trim. Для Gmail/Googlemail: удаление точек в локальной части перед @ (специфика Google\!). Телефон: формат E.164 с \+. |
| **TikTok Events API (v1.3 / 2.0)** | REST (JSON). Auth: Access Token (Events Manager). | До **1 000** событий в батче. Rate limit: \~50–100 req/sec на токен \[UNVERIFIED — зависит от tier\]. | **7 дней** (для оптимизации; старше могут приниматься в статистику с ограничениями или отклоняться). | По event\_id \+ event (имя события). Окно дедупликации — **48 часов**. | Lowercase, trim. Телефон: E.164 (с \+ или без — в зависимости от версии, стандарт E.164). |
| **Snapchat Conversions API (v2/v3)** | REST (JSON). Auth: Bearer Token (OAuth / Static CAPI token). | До **2 000** событий в батче. | **7 дней** (до 28 дней для оффлайн-импорта, но для web CAPI — 7 дней). | По client\_dedup\_id (окно **48 часов**). | Lowercase, trim. Телефон: только цифры с кодом страны, без ведущих нулей и \+. |
| **Microsoft Ads (Conversions API / Offline)** | REST / SOAP. Auth: OAuth 2.0 \+ Developer Token. | До **1 000** записей в батче (зависит от эндпоинта). | До **90 дней** от клика msclkid. | По msclkid \+ ConversionName \+ ConversionTime (или OrderId). | Lowercase, trim. Телефон: E.164. |

**Ключевые изменения 2025–2026 годов (факт):**

* **Google Data Manager API:** Google активно консолидирует загрузку оффлайн-конверсий, аудиторий и Enhanced Conversions в единый Data Manager API, постепенно упрощая работу с gclid/wbraid/gbraid и требуя обязательной передачи сигналов согласия (consent object) для трафика из EEA/UK.  
* **Частичные ошибки (Partial Failures):** Meta CAPI и Google Ads API поддерживают partial\_failure=true — запрос возвращает HTTP 200 OK, но внутри JSON-ответа часть событий может быть отклонена (например, невалидный хеш или event\_time старше 7 дней). **Критическая ошибка** — считать HTTP 200 успехом всего батча без парсинга тела ответа.  
* **Тестовые инструменты:** Meta Test Events (test\_event\_code), TikTok Test Event Code, Google Ads validate\_only=true.

### **3.2. Маппинг вымышленных сетей кейса (A, B, C) на реальные аналоги**

| Сеть из кейса | Ближайший реальный аналог | Что реально (индустрия) | Что является спецификой кейса |
| :---- | :---- | :---- | :---- |
| **Сеть A** | **Meta Conversions API** (и частично TikTok Events API) | Батч до 1 000 событий, дедупликация по event\_id в окне 48 часов, лимит возраста 7 дней, SHA-256 для email. | Жесткий фиксированный лимит 100 req/min (у Meta лимит динамический, но 100 req/min — удобное и реалистичное ограничение для проектирования rate limiter). |
| **Сеть B** | **Устаревшие / нишевые DSP, нативные сети (Taboola/Outbrain/Push-сети) или легаси-эндпоинты** | 1 событие на запрос, отсутствие дедупликации на стороне приемника, нестабильный HTTP с таймаутами, наличие дневного отчета (CSV/API). | Комбинация «частые ложные таймауты \+ жесткий запрет на дубли \+ только дневной отчет» — классический стресс-тест на проектирование распределенных систем с неидемпотентным внешним API. |
| **Сеть C** | **Affiliate/CPA-платформы** (Everflow, CAKE, HasOffers/TUNE, Impact, Affise, Scaleo) и трекеры (Voluum, Keitaro, RedTrack) | S2S GET postback с click\_id и суммой, оплата за первый депозит (FTD), постбэк отмены/чарджбэка, ежемесячные сверки (invoicing/reconciliation). | Простои эндпоинта на несколько часов (в реальности бывает у небольших партнерских программ и self-hosted трекеров вроде Keitaro). |

### **3.3. Паттерны Affiliate S2S Postback (индустриальная практика)**

Как работают реальные партнерские трекеры (Voluum, RedTrack, Everflow, Affise, Impact):

* **Формат:** HTTP GET на URL вида:  
  \[https\://partner-tracker.com/postback?click\_id=\](https\://partner-tracker.com/postback?click\_id=){click\_id}\&payout={amount}\&currency={currency}\&txid={transaction\_id}\&status={status}  
* **Идемпотентность:** Продвинутые трекеры поддерживают параметр txid (Transaction ID) — повторный запрос с тем же (click\_id, txid) не создает новую конверсию. Если txid не поддерживается, дедупликация идет по (click\_id, goal/event\_type).  
* **Чарджбэки / Отмены (Reversals):** Передаются либо отдельным эндпоинтом, либо тем же URL со статусом status=rejected / status=chargeback / status=cancelled и тем же txid (или click\_id). В финансовых и iGaming-вертикалях часто используется **Hold Period** (период холда 14–30 дней до выплаты), в течение которого статус может смениться с pending на approved или rejected.  
* **Безопасность:** Передача статического api\_key/token в query/header, IP-whitelisting серверов отправителя или HMAC-подпись параметров.

## **4\. Рекомендуемая архитектура**

### **4.1. Оценка нагрузки (Capacity Check)**

Прежде чем выбирать архитектуру, переведем требования в реальные цифры:

* **Базовый объем:** 50 000 регистраций/день \+ 5 000 первых депозитов/день \= **55 000 событий/день**.  
* **Средняя нагрузка:** $55000/86400\approx 0.64событий/сек$.  
* **Пиковая нагрузка (5× на промо/стримах):** $0.64\times 5\approx 3.2событий/сек$ (\~192 события/мин).  
* **Даже с учетом локальных всплесков (10× от пика в течение 1 минуты):** \~30 событий/сек.

**Нагрузка по сетям (в предположении, что 100% трафика идет в каждую сеть — верхняя граница):**

* **Сеть A (батч до 1000, лимит 100 req/min):** При отправке батчей каждые 5–10 секунд достаточно **6–12 запросов в минуту** (используем менее 12% лимита).  
* **Сеть B (1 событие \= 1 запрос):** В пике \~3.2 req/sec (192 req/min). Легко обслуживается одним небольшим воркером с пулом из 10–20 горутин/корутин (с учетом таймаутов).  
* **Сеть C (только депозиты \+ чарджбэки):** 5 000 депозитов/день \= **0.06 событий/сек** в среднем, **\~0.3 событий/сек** в пике.

**Архитектурный вывод:** Это **НЕ задача про Highload**. Здесь не нужен сложный шардинг, Flink или десятки микросервисов. Вся сложность лежит в плоскости **надежности, корректности, идемпотентности, обработки частичных сбоев и сверки денег**.

### **4.2. Компонентная архитектура**

Фрагмент кода

```
flowchart TB
    subgraph Client["Фронтенд / Лендинги (и зеркала)"]
        FE[JS Attribution SDK + GTM]
    end

    subgraph Core["Существующий PHP-монолит"]
        PHP[PHP-FPM App]
        DB[(MariaDB: Core + Outbox)]
    end

    subgraph TrackingPlatform["Conversion Tracking Platform (Новый сервис)"]
        AttrAPI[Attribution API]
        AttrDB[(MariaDB / Redis: Attribution Store & Delivery Ledger)]
        Relay[Outbox Relay / Debezium]
        Kafka[(Kafka: domain.conversions)]
        Router[Enrichment & Router Worker]
        
        subgraph Adapters["Per-Network Delivery Workers"]
            AdA[Adapter A: Batch + Rate Limiter]
            AdB[Adapter B: Strict No-Blind-Retry]
            AdC[Adapter C: Payout Ledger + Long Retry]
        end
        
        Recon[Reconciliation Jobs: Daily B / Monthly C]
        Obs[Metrics & Marketing Reporting View]
    end

    FE -- "1. POST /touchpoint (click_ids, consent)" --> AttrAPI
    AttrAPI -- "Сохраняет сессию, возвращает attr_token" --> AttrDB
    FE -- "2. Регистрация + attr_token + event_id" --> PHP
    PHP -- "3. Транзакция: бизнес-данные + Outbox" --> DB
    DB -- "4. Чтение Outbox" --> Relay
    Relay -- "5. Publish" --> Kafka
    Kafka --> Router
    Router -- "6. Обогащение (Click IDs, SHA-256 email, consent)" --> AttrDB
    Router -- "7. Запись в Delivery Ledger (PENDING)" --> AttrDB
    
    AttrDB --> AdA & AdB & AdC
    AdA -- "REST Batch (<=1000)" --> NetA((Сеть A))
    AdB -- "REST 1-by-1" --> NetB((Сеть B))
    AdC -- "GET S2S Postback" --> NetC((Сеть C))
    
    NetB -. "Daily Report" .-> Recon
    NetC -. "Monthly Export" .-> Recon
    Recon --> AttrDB
    AttrDB --> Obs
```

### **4.3. Стейт-машина доставки (Delivery State Machine)**

Каждая пара (conversion\_event\_id, network\_id) имеет независимый жизненный цикл в таблице delivery\_ledger:

Фрагмент кода

```
stateDiagram-v2
    [*] --> PENDING: Событие обогащено и прошло фильтр (consent + attribution)
    PENDING --> SENDING: Воркер взял в работу (lease / lock)
    
    SENDING --> ACCEPTED: HTTP 2xx и подтверждение в теле ответа
    SENDING --> RETRY_SCHEDULED: Временная ошибка (5xx, 429, network reset) [Сети A, C]
    SENDING --> UNKNOWN: Таймаут / обрыв после отправки запроса [Сеть B]
    SENDING --> REJECTED: Перманентная ошибка (4xx, невалидный хеш, дубль)
    
    RETRY_SCHEDULED --> SENDING: Наступило next_retry_at
    RETRY_SCHEDULED --> EXPIRED: Превышен event_age_limit (напр. 7 дней для A)
    RETRY_SCHEDULED --> DEAD_LETTER: Исчерпаны попытки (ручной разбор)
    
    UNKNOWN --> ACCEPTED: Найдено в Daily Report (Reconciliation)
    UNKNOWN --> RETRY_SCHEDULED: НЕ найдено в Daily Report (безопасный ретрай)
    UNKNOWN --> MANUAL_REVIEW: Неоднозначное совпадение в отчете
```

### **4.4. Детальные решения и отвергнутые альтернативы**

| Область | Рекомендуемое решение | Отвергнутые альтернативы | Почему выбрано и компромиссы |
| :---- | :---- | :---- | :---- |
| **1\. Захват Click ID и связь с монолитом** | **Attribution Service \+ 1 непрозрачный attribution\_token**. Фронт на первом хите шлет все параметры URL и consent в Attribution API, получает attr\_token (хранится в 1st-party cookie \+ localStorage). При регистрации фронт передает attr\_token и event\_id в PHP-монолит. Монолит просто сохраняет их в событие. | 1\) Хранить все Click ID прямо в таблицах PHP-монолита. 2\) Использовать только GTM без бэкенда. | **Минимальное изменение монолита:** мейнтейнерам PHP нужно добавить лишь проброс 2 строковых полей (attr\_token, event\_id). Новые сети добавляются без правок PHP. Компромисс: нужен легкий сервис приема хитов. |
| **2\. Зеркала доменов (Mirror Domains)** | **Link Decoration (проброс ?at=\<attr\_token\> при редиректе между зеркалами)** \+ **Server-set HttpOnly cookie** через единый трекинговый субдомен (если возможно) или централизованный редиректор. | Полагаться на 3rd-party cookies. | 3rd-party cookies заблокированы в Safari/Firefox. Если пользователь переходит по нашей ссылке на зеркало — декорируем URL токеном. Если вводит зеркало руками — атрибуция невосстановима на клиенте, но восстанавливается по user\_id после логина. |
| **3\. Извлечение событий из PHP-монолита** | **Transactional Outbox в MariaDB** \+ отдельный легковесный Relay-процесс (polling outbox\_events каждые 1–2 сек) → публикация в Kafka. | 1\) Прямая отправка в Kafka из PHP-FPM. 2\) CDC (Debezium) на binlog MariaDB. | **Прямая отправка из PHP-FPM** опасна: нет транзакционности с БД (dual-write), а librdkafka в короткоживущих PHP-воркерах создает проблемы с коннектами. **Debezium на MariaDB** требует настройки ROW binlog, имеет нюансы с MariaDB GTID (отличается от MySQL GTID) и избыточен для 3 событий/сек. Outbox прост, надежен и понятен команде. |
| **4\. Гарантии доставки и ретраи** | **At-least-once из Kafka \+ DB-backed Delivery Ledger** (MariaDB) с next\_retry\_at, exponential backoff \+ jitter. | 1\) Kafka Exactly-Once Semantics (EOS). 2\) Отдельные Retry-топики в Kafka для задержек на часы. | **Kafka EOS не работает для внешних HTTP-вызовов** (оно гарантирует транзакционность только внутри Kafka: read-process-write). Retry-топики в Kafka плохо подходят для задержек на 12–48 часов (как для сети C и сверки сети B). Таблица в БД дает полный контроль, аудит, легкий replay и готовые данные для дашборда маркетинга. |
| **5\. Модель атрибуции (Multi-touch)** | Хранить **цепочку касаний** (attribution\_touchpoints), но для отправки в сети использовать **Last Non-Direct Click** (в пределах окна, напр. 30 дней) для каждого канала или общий Last Click (согласовать с маркетингом). | Перезаписывать единственный Click ID в cookie. | Хранение истории касаний позволяет менять правила атрибуции и разбирать споры с аффилиатами (например, органический трафик не должен перетирать платный клик). |
| **6\. Build vs Buy (sGTM / CDP)** | **Кастомный легковесный сервис** (Go / Node / Python / PHP CLI) на базе существующих MariaDB \+ Kafka. | 1\) Server-side GTM (sGTM / Stape). 2\) Segment / RudderStack. | sGTM отлично подходит для стандартных Meta/Google CAPI, но **плохо решает задачи сети B** (карантин UNKNOWN, сверка с дневным отчетом) и **сети C** (финансовый Payout Ledger, чарджбэки, месячная сверка). Кастомный сервис на 3 адаптера при 3 req/sec прост в поддержке и дает 100% контроль. |

### **4.5. Схема данных (MariaDB-friendly)**

SQL

```
-- 1. Outbox в БД PHP-монолита (минимальное изменение)
CREATE TABLE conversion_outbox (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_id        CHAR(36) NOT NULL,               -- UUID (совпадает с пикселем!)
    event_type      ENUM('REGISTRATION', 'FIRST_DEPOSIT', 'CHARGEBACK') NOT NULL,
    user_id         BIGINT UNSIGNED NOT NULL,
    attr_token      CHAR(36) NULL,                   -- Непрозрачный токен от фронтенда
    payload_json    JSON NOT NULL,                   -- amount, currency, tx_id, occurred_at
    created_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    published_at    DATETIME(3) NULL,
    UNIQUE KEY uk_event_id (event_id),
    KEY idx_unpublished (published_at, id)
);

-- 2. Хранилище атрибуции (в сервисе трекинга)
CREATE TABLE attribution_sessions (
    attr_token      CHAR(36) PRIMARY KEY,
    user_id         BIGINT UNSIGNED NULL,            -- Привязывается при регистрации
    network_hint    VARCHAR(32) NULL,                -- 'net_a', 'net_b', 'net_c'
    click_ids_json  JSON NOT NULL,                   -- {"gclid": "...", "fbclid": "...", "click_id": "..."}
    client_meta     JSON NOT NULL,                   -- ip, user_agent, fbp, fbc, landing_url
    consent_state   JSON NOT NULL,                   -- {"ad_storage": "granted", "ad_user_data": "granted"}
    created_at      DATETIME(3) NOT NULL,
    expires_at      DATETIME(3) NOT NULL,            -- TTL для GDPR (напр. 90 дней)
    KEY idx_user_id (user_id),
    KEY idx_expires (expires_at)
);

-- 3. Журнал доставок (Delivery Ledger — сердце системы и источник отчетов)
CREATE TABLE delivery_ledger (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_id            CHAR(36) NOT NULL,
    network_id          ENUM('NET_A', 'NET_B', 'NET_C') NOT NULL,
    event_type          ENUM('REGISTRATION', 'FIRST_DEPOSIT', 'CHARGEBACK') NOT NULL,
    user_id             BIGINT UNSIGNED NOT NULL,
    occurred_at         DATETIME(3) NOT NULL,        -- Бизнес-время события (для SLO 15 мин и лимита 7 дней)
    status              ENUM('PENDING','SENDING','ACCEPTED','RETRY_SCHEDULED','UNKNOWN','REJECTED','EXPIRED','DEAD_LETTER') NOT NULL,
    attempt_count       SMALLINT UNSIGNED NOT NULL DEFAULT 0,
    next_retry_at       DATETIME(3) NULL,
    external_ref_id     VARCHAR(128) NULL,           -- ID ответа сети или tx_id
    reject_reason_code  VARCHAR(64) NULL,            -- Таксономия ошибок для маркетинга
    last_http_status    SMALLINT NULL,
    last_error_detail   VARCHAR(512) NULL,
    updated_at          DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3),
    UNIQUE KEY uk_event_network (event_id, network_id),
    KEY idx_poll_queue (network_id, status, next_retry_at),
    KEY idx_recon_lookup (network_id, status, occurred_at)
);
```

**Что хранить в Kafka, а что в MariaDB:**

* **Kafka** — транспортная шина для развязки монолита и трекинга (буфер на случай пиков и деплоев, ретеншн 7 дней).  
* **MariaDB** — источник истины (source of truth) для состояний доставки, финансового леджера аффилиатов, расписания длинных ретраев и аналитических выборок маркетинга.

### **4.6. Наблюдаемость (Observability) и отчетность для маркетинга**

* **SLO 15 минут:** Измеряется гистограммой conversion\_e2e\_latency\_seconds \= delivered\_at \- occurred\_at (от времени бизнес-события в БД, а не от момента попадания в воркер). Алерт при p95 \> 10 мин или p99 \> 15 мин.  
* **Таксономия причин отказов (reject\_reason\_code):** Маркетингу не нужны HTTP-коды, им нужны понятные категории:  
  * NO\_CONSENT — пользователь не дал согласие.  
  * NO\_CLICK\_ID — органический трафик или потерянный Click ID.  
  * EVENT\_TOO\_OLD — превышен лимит возраста сети (например, более 7 дней для A).  
  * INVALID\_PII — невалидный формат email/телефона.  
  * NETWORK\_REJECTED — отклонено валидацией сети (с текстом ошибки).  
  * TIMEOUT\_UNKNOWN — зависло в статусе UNKNOWN (Сеть B), ждет сверки.  
* **Витрина для маркетинга:** Простой SQL-view или материализованная таблица (обновляемая раз в 5 минут) поверх delivery\_ledger, подключенная к существующему BI (Metabase / Looker / Grafana) с разбивкой по дням, сетям и статусам: Sent / Accepted / Rejected (by reason) / Pending / Unknown.

### **4.7. Безопасный выкат (Rollout)**

1. **Shadow Mode (Режим тени):** Включаем сбор attribution\_token и заполнение delivery\_ledger, но вместо реального HTTP-вызова пишем в лог (dry-run). Сравниваем объем событий с GTM-пикселем (ожидаем увидеть \+20–40% событий, которые не дошли через браузер).  
2. **Проверка дедупликации (Сеть A):** Используем тестовый режим сети (test\_event\_code), проверяем, что event\_id в браузере и на сервере совпадают **до символа**.  
3. **Canary по сетям:** Включаем продакшн-отправку по одной сети (сначала A — она безопасна благодаря дедупликации, затем C — с контролем на одном партнере, последней B — после обкатки парсера дневных отчетов).  
4. **Правила Backfill:** Запрещено автоматически досылать старые события, возраст которых превышает event\_age\_limit сети (для A — 7 дней).

## **5\. Разбор «трудных» сетей**

### **5.1. Сеть B: таймауты без дедупликации \+ дневной отчет**

**Суть проблемы:** При таймауте клиент не знает, обработал ли сервер запрос. Повторная отправка создает дубль, который ломает ML-оптимизацию ставок.

**Компромисс «Скорость vs Дубликаты»:**

* Если ретраить сразу — получаем высокую скорость, но ломаем обучение сети дублями (неприемлемо по условиям кейса).  
* Если не ретраить никогда — теряем реальные конверсии при сетевых обрывах.  
* **Оптимальное правило (Decision Rule):** При ошибке соединения **до** отправки байтов (connection refused, DNS error) — ретраить безопасно. При **таймауте чтения ответа** (read timeout, connection reset after write) — переводить в статус UNKNOWN и **ждать дневной отчет**.

**Алгоритм обработки и сверки (псевдокод):**

Python

```
# 1. Отправка события в Сеть B
def send_to_network_b(record):
    try:
        # Таймаут ставим щедрым (напр. 15-20 сек), чтобы снизить число ложных UNKNOWN
        resp = http_post(NET_B_URL, json=record.payload, timeout=(3.0, 20.0))
        if resp.status_code == 200:
            mark_status(record.id, 'ACCEPTED')
        elif resp.status_code in (400, 422):
            mark_status(record.id, 'REJECTED', reason=resp.text)
        elif resp.status_code == 429:
            schedule_retry(record.id, backoff_with_jitter())
        else:
            # 502/503/504 — могли успеть обработать на стороне сети
            mark_status(record.id, 'UNKNOWN', detail=f"HTTP {resp.status_code}")
    except ConnectTimeoutError:
        # Соединение не установилось, запрос не ушел — безопасно ретраить
        schedule_retry(record.id, backoff_with_jitter())
    except ReadTimeoutError:
        # Запрос ушел, ответ не пришел — НИКАКИХ РЕТРАЕВ!
        mark_status(record.id, 'UNKNOWN', detail="Read timeout")

# 2. Ежедневная сверка (запускается после публикации Daily Report)
def reconcile_network_b(daily_report_rows):
    unknowns = db.fetch("SELECT * FROM delivery_ledger WHERE network_id='NET_B' AND status='UNKNOWN'")
    
    for rec in unknowns:
        # Вариант 1 (идеальный): отчет содержит наш event_id или click_id
        if match_exact(rec, daily_report_rows):
            mark_status(rec.id, 'ACCEPTED', detail="Confirmed via daily report")
        # Вариант 2 (отчет без нашего ID): вероятностное сопоставление
        elif match_probabilistic(rec, daily_report_rows, time_window_sec=60):
            # Совпали: click_id + event_type + сумма + время (±60 сек)
            mark_status(rec.id, 'ACCEPTED', detail="Probabilistic match")
        else:
            # События точно нет в отчете — теперь ретрай безопасен!
            if age_in_days(rec.occurred_at) < MAX_SAFE_AGE_DAYS:
                schedule_retry(rec.id, immediate=True)
            else:
                mark_status(rec.id, 'EXPIRED')
```

**Edge case (неоднозначное совпадение):** Если отчет не содержит event\_id, и за одну минуту произошло 2 одинаковых депозита по одному click\_id, а в отчете только 1 — не ретраить автоматически; перевести в MANUAL\_REVIEW и поднять алерт.

### **5.2. Сеть C: аффилиат S2S, простои, чарджбэки и месячная сверка**

**Суть проблемы:** Прямые деньги. Нужна гарантия «ровно одна выплата за первый депозит», устойчивость к многочасовым простоям, обязательная отправка чарджбэка и автоматизация ежемесячной сверки.

**Архитектурные механизмы:**

1. **Финансовый Payout Ledger:** Отдельная таблица affiliate\_payout\_ledger с уникальным ключом UNIQUE (partner\_id, click\_id, event\_type) (для FIRST\_DEPOSIT). Даже если из Kafka придет дубль депозита, БД физически не даст создать вторую запись на выплату.  
2. **Устойчивость к простоям (часы):** Exponential backoff с потолком (cap) в 15–30 минут и горизонтом ретраев **до 72 часов** (плюс Circuit Breaker: если 10 запросов подряд упали с 5xx/timeout, воркер переходит в режим редкого прощупывания раз в 5 минут, чтобы не забивать логи и пул соединений).  
3. **Порядок и Чарджбэки (Ordering):**  
   * Партиционирование в Kafka по user\_id гарантирует, что CHARGEBACK придет после FIRST\_DEPOSIT.  
   * Если FIRST\_DEPOSIT еще не доставлен (например, Сеть C лежит), а уже пришел CHARGEBACK: **не отправлять ни депозит, ни чарджбэк** — отменить депозит локально (статус CANCELLED\_BEFORE\_SEND).  
   * Если FIRST\_DEPOSIT уже в статусе ACCEPTED — отправить postback отмены (status=rejected\&txid=\<deposit\_tx\_id\>).  
4. **Автоматическая ежемесячная сверка (Reconciliation):**  
   Скрипт загружает CSV/Excel-выгрузку партнера и делает FULL OUTER JOIN с affiliate\_payout\_ledger по click\_id (и tx\_id, если поддерживается):  
   * MATCHED — суммы совпадают (допуск на округление FX ±0.01).  
   * MISSING\_AT\_PARTNER — у нас ACCEPTED, у партнера нет (проверить логи HTTP 200, возможно сбой на их стороне).  
   * EXTRA\_AT\_PARTNER — у партнера есть, у нас нет или был CHARGEBACK (основание для оспаривания инвойса\!).  
   * AMOUNT\_MISMATCH — расхождение суммы или валюты.

## **6\. Приватность и согласие (2026)**

Чек-лист соответствия требованиям GDPR, ePrivacy, DMA (Digital Markets Act) и политикам рекламных платформ:

* \[ \] **Проброс состояния согласия (Consent Propagation):** Каждый запрос от фронтенда в Attribution Service обязан содержать актуальный объект согласия (например, флаги Google Consent Mode v2: ad\_storage, ad\_user\_data, ad\_personalization, а также IAB TCF v2.2 string для EEA/UK).  
* \[ \] **Жесткий гейт на сервере (Server-Side Enforcement):** Если пользователь из EEA/UK не дал согласие (ad\_user\_data \= denied), сервер **не имеет права** отправлять его хешированный email/телефон или Click ID в рекламные сети (кроме строго оговоренных законом случаев или агрегированного моделирования без PII).  
* \[ \] **Правовое основание для хешированных PII:** SHA-256 хеш email — это **псевдонимизированные персональные данные (PII)** по GDPR, а не анонимные данные (факт, разъяснения EDPB). Их передача требует согласия пользователя (Consent) и подписанного DPA (Data Processing Agreement) с платформой.  
* \[ \] **Минимизация и TTL данных:** Хранить Click ID и метаданные сессии не дольше, чем требует бизнес-логика и окно атрибуции (обычно 90 дней для атрибуции; финансовый леджер аффилиатов — по срокам бухгалтерского учета, но без лишних PII).  
* \[ \] **Ограничения для чувствительных вертикалей (флаг риска):** Наличие «депозитов / пополнения баланса» характерно для финтеха, криптосервисов и iGaming. В 2025–2026 годах Meta, Google и TikTok жестко контролируют передачу данных из таких категорий (Restricted Data / Special Ad Categories).  
* \[ \] **Что НИКОГДА нельзя отправлять в рекламные сети:**  
  * Сырые (нехешированные) email, телефоны, ФИО.  
  * Номера банковских карт, паспортные данные, внутренние балансы счетов, признаки уязвимости или медицинские данные.  
  * Точные URL страниц, содержащие PII в query-параметрах (например, /confirm?email=user@mail.com).

## **7\. Типичные сбои и как их избежать**

| Сбой (Failure Mode) | Симптом в продакшне | Предотвращение (Prevention) | Мониторинг (Monitoring) |
| :---- | :---- | :---- | :---- |
| **Рассинхронизация event\_id** | В кабинете сети конверсий стало в 2 раза больше после запуска S2S. | Генерировать event\_id (UUIDv4) в одном месте (на фронте при отправке формы или на бэке с возвратом на фронт) и передавать **один и тот же ID** в GTM и в Outbox. | Сравнение server\_accepted vs pixel\_fired и проверка метрики Deduplication Rate в интерфейсе сети. |
| **Partial Failure \= HTTP 200** | Система рапортует 100% успеха, но в рекламном кабинете нет 30% событий. | Всегда парсить тело ответа батча (массивы errors / partial\_failure\_error) и обновлять статус в delivery\_ledger поштучно. | Алерт на рост доли REJECTED внутри батчей; логирование структуры ответов. |
| **Ошибки нормализации перед SHA-256** | Низкий Event Match Quality (EMQ менее 5/10), сеть не находит пользователей. | Строгий unit-tested нормализатор под каждую сеть (trim, lowercase, E.164 для телефонов, удаление точек в Gmail для Google). | Регулярная проверка EMQ/Match Rate в API диагностики сети; unit-тесты на edge cases. |
| **Clock Skew / Timezone bug** | Массовые отказы event\_time in the future или event too old. | Использовать строго UTC (Unix timestamp в секундах/миллисекундах) во всей системе; NTP-синхронизация серверов. | Алерт на reject\_reason\_code \= EVENT\_TOO\_OLD или INVALID\_TIMESTAMP. |
| **Потеря чарджбэка** | Компания платит аффилиатам за отмененные/фродовые депозиты. | Чарджбэк обязан попадать в тот же Outbox, что и депозит; ежемесячная авто-сверка с выгрузкой партнера. | Метрика chargebacks\_sent / chargebacks\_total (должна быть 100% для партнерского трафика). |
| **Отправка до получения Consent** | Жалобы DPO, предупреждения или бан рекламного аккаунта. | Проверять consent\_state в роутере перед постановкой в delivery\_ledger; тесты на дефолтный отказ (fail-closed). | Аудит-метрика: 0 событий отправлено с consent \= denied для EEA/UK. |

## **8\. Допущения и открытые вопросы (готово для RFC)**

| ID | Допущение (Assumption) | Зачем нужно для дизайна | Как проверить (валидация) | Риск, если неверно |
| :---- | :---- | :---- | :---- | :---- |
| **A-01** | Сеть A дедуплицирует по event\_id \+ event\_name и возвращает детальный статус по каждому событию в батче. | Позволяет безопасно ретраить батчи и вести точный учет accepted/rejected. | Изучить API-документацию Сети A и прогнать тест в sandbox. | Придется снижать размер батча или менять логику обработки ошибок. |
| **A-02** | Дневной отчет Сети B содержит либо наш event\_id, либо пару (click\_id, timestamp, amount). | Критично для точного выхода из статуса UNKNOWN без дублей. | Запросить пример CSV/API-отчета у аккаунт-менеджера Сети B. | Если в отчете только агрегаты (итого за день), точная сверка невозможна — придется согласовать с бизнесом политику потерь (не ретраить UNKNOWN). |
| **A-03** | Сеть C поддерживает параметр txid (или дедуплицирует по click\_id \+ event\_type) для идемпотентности. | Позволяет безопасно ретраить GET postback при таймаутах Сети C. | Уточнить у технической поддержки Сети C; протестировать двойной вызов. | Если идемпотентности нет, к Сети C придется применить строгую схему UNKNOWN (как к Сети B). |
| **A-04** | Сеть C поддерживает S2S postback для отмены/чарджбэка (например, status=rejected). | Позволяет автоматизировать отмену выплат при чарджбэке. | Проверить спецификацию постбэков Сети C. | Чарджбэки придется выгружать и оспаривать вручную на ежемесячной сверке. |
| **A-05** | При переходах между зеркалами доменов мы контролируем редиректы или ссылки. | Позволяет пробрасывать attr\_token через URL (Link Decoration). | Уточнить у DevOps/Frontend, как устроен роутинг зеркал. | При прямом ручном переходе на зеркало до регистрации атрибуция будет потеряна. |
| **A-06** | В PHP-монолите допустимо добавить 1 таблицу conversion\_outbox и проброс 2 полей (attr\_token, event\_id). | Базовое условие для минимизации изменений в монолите. | Согласовать RFC с техлидом PHP-монолита. | При полном запрете правок БД монолита придется использовать CDC (Debezium) на существующие таблицы users/deposits. |
| **A-07** | Платформа работает с пользователями из EEA/UK (действуют GDPR/DMA и Consent Mode v2). | Определяет необходимость жесткого серверного гейта по согласию. | Уточнить географию трафика у Legal/Marketing. | Если трафик только вне EEA, требования мягче, но архитектура с флагом согласия остается хорошей практикой. |

## **9\. Мини-модуль для второй части задания (AI-агенты, процессы и метрики)**

### **9.1. Управление кодом, сгенерированным AI-агентами (2026)**

* **Risk-Tiered Review:** Разделение изменений на уровни риска. *Tier 1 (низкий — лендинги, верстка, A/B-тесты):* авто-проверки \+ быстрое ревью. *Tier 2 (высокий — трекинг, PII, финансовый леджер, правки в чужих репозиториях):* обязательное ревью senior-инженером.  
* **Инструкции для агентов (AGENTS.md / CLAUDE.md):** В корне репозитория фиксируются архитектурные границы, запреты (например, «никогда не логировать PII», «не менять схемы БД без миграции»), команды запуска тестов и стиль кода.  
* **Обязательные CI-гейты (необсуждаемые):** SAST (Semgrep/CodeQL), SCA и проверка лицензий/CVE (Dependabot/Renovate \+ авто-генерация SBOM в формате CycloneDX/SPDX), сканирование секретов (Gitleaks/TruffleHog).  
* **Изоляция и права агентов:** Агенты работают в песочницах без прямого доступа к продакшн-секретам и без права push в main.  
* **Контрибьюция в чужие репозитории (PHP-монолит):** Агент готовит минимальный, атомарный PR строго по шаблону и линтерам команды-владельца; перед отправкой PR обязательно проходит внутреннее ревью в Marketing Engineering, чтобы не нагружать чужих мейнтейнеров «AI-шумом».

### **9.2. Приоритизация запросов (Intake) и метрики команды**

* **Приоритизация:** Модель **WSJF (Weighted Shortest Job First)** / Cost of Delay — оцениваем, сколько денег теряет маркетинг каждую неделю без задачи (например, потеря 30% конверсий стоит дороже, чем новый виджет на лендинге). **Жесткая квота 20–25% емкости** резервируется под техдолг, безопасность, обновление API сетей и поддержку.  
* **Метрики эффективности (Outcome-oriented):**  
  * *Бизнес-качество трекинга:* Tracking Coverage (доля конверсий с успешной доставкой), Event Match Quality (EMQ), расхождение сверки с аффилиатами (Discrepancy Rate %).  
  * *Инженерная эффективность (адаптированный DORA):* Lead Time for Changes (время запуска новой кампании/интеграции), Change Failure Rate (инциденты с потерей/дублированием данных), SLO доставки (доля событий, доставленных менее чем за 15 минут).

## **10\. План для RFC и вопросы ревьюеров**

### **Рекомендуемая структура RFC (на 1.5 часа работы)**

1. **Context & Problem Statement:** Текущие потери 20–40%, влияние на ставки и споры с аффилиатами, цели (гибрид, SLO 15 мин, прозрачность).  
2. **Non-Goals & Constraints:** Мы не переписываем PHP-монолит, не заменяем GTM-пиксель, не строим полноценный CDP.  
3. **Capacity & Scale:** Расчет \~3.2 событий/сек в пике (обоснование простоты архитектуры).  
4. **High-Level Architecture:** Компонентная диаграмма \+ поток данных (Click → Token → Outbox → Kafka → Delivery Ledger → Adapters).  
5. **Deep Dive: Network-Specific Challenges:**  
   * Сеть A (батчинг, rate limit 100 req/min, дедупликация 48ч, лимит 7 дней).  
   * Сеть B (статус UNKNOWN, запрет слепых ретраев, дневная сверка).  
   * Сеть C (Payout Ledger, простои, чарджбэки, месячная сверка).  
6. **Data Model & Monolith Impact:** Ровно 1 таблица Outbox \+ 2 поля при регистрации (почему это безопасно для PHP-команды).  
7. **Privacy, Security & Observability:** Consent gate, SHA-256, метрики, витрина для маркетинга.  
8. **Alternatives Considered & Rollout Plan:** Почему не sGTM/Debezium; Shadow Mode → Canary.  
9. **Assumptions & Open Questions:** Таблица из раздела 8\.

### **Чек-лист вопросов ревьюеров, которые RFC должен закрыть заранее**

* *«Почему нельзя просто поставить Server-side GTM?»* → Ответ: sGTM не решает карантин UNKNOWN с дневной сверкой (Сеть B) и финансовый леджер с чарджбэками (Сеть C).  
* *«Что будет, если Kafka упадет на час?»* → Ответ: События безопасно копятся в conversion\_outbox в MariaDB и отправятся после восстановления.  
* *«Как мы гарантируем, что не сломаем статистику дублями в день запуска?»* → Ответ: Shadow mode \+ сверка event\_id в тестовом режиме сети до включения продакшна.  
* *«Что делать, если пользователь зарегистрировался на зеркале?»* → Ответ: Link decoration при редиректе; если зашел руками — честное признание ограничения (невозможно без нарушения Same-Origin Policy).

## **11\. Глоссарий**

* **S2S Postback (Server-to-Server):** Прямой HTTP-запрос с сервера рекламодателя на сервер партнерской/рекламной сети для уведомления о конверсии без участия браузера.  
* **CAPI (Conversions API):** Серверный API рекламных платформ (Meta, TikTok, Snap) для передачи веб\- и оффлайн-событий напрямую с бэкенда.  
* **Offline Conversions:** Конверсии, происходящие вне браузера (в CRM, платежном шлюзе, колл-центре) и загружаемые в рекламную сеть с задержкой.  
* **Enhanced Conversions:** Технология Google Ads, дополняющая теги конверсий хешированными first-party данными (email, телефон) для повышения точности атрибуции.  
* **Advanced Matching:** Передача хешированных данных пользователя в Meta (через пиксель или CAPI) для сопоставления с аккаунтом в соцсети.  
* **Deduplication (Дедупликация):** Механизм отбрасывания дубликатов одного и того же события, пришедшего из браузера и с сервера, на основе общего event\_id.  
* **Attribution Window / Lookback Window:** Период времени после клика или показа (например, 7 или 30 дней), в течение которого конверсия засчитывается данной рекламной кампании.  
* **Click-Through vs View-Through:** Атрибуция конверсии клику по рекламе (Click-Through) против атрибуции только факту просмотра рекламы без клика (View-Through).  
* **MMP (Mobile Measurement Partner):** Независимый трекер мобильной атрибуции (AppsFlyer, Adjust, Singular), связывающий установки приложений с рекламными сетями.  
* **Ad Network vs Affiliate Platform vs Tracker:** Рекламная сеть продает показы/клики через аукцион (Meta, Google); аффилиат-платформа связывает рекламодателя с вебмастерами за CPA/RevShare (Everflow, Impact); трекер — инструмент учета кликов и маршрутизации трафика (Voluum, Keitaro).

