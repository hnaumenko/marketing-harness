# Assumptions Register

Maintained by the `assumptions` mode. Statuses: `open`, `confirmed`, `narrative-only`, `user-cannot-confirm`.
`user-cannot-confirm` is permanent: re-citing a claim never upgrades it.
Markers: see `AGENTS.md`. `design-decision` = the author's proposal, not a fact; it needs reviewer approval.
None of the rows below has been through the four-outcome confirmation yet.

| id | claim | marker | source (URL, date) | status |
|---|---|---|---|---|
| A-001 | Событие уходит только в сеть, чей Click ID стоит в атрибуции. Альтернатива из Gemini: слать в A и B события всех пользователей с хэшем email ради сопоставления | design-decision | README §5.3; альтернатива: Gemini PDF §4.4 | open |
| A-002 | Сеть A возвращает результат по каждому событию батча; иначе при HTTP 400 делим батч пополам | derived-unverified | docs/claude-deep-research.md §8 A-01 | open |
| A-003 | Сеть A убирает дубли и между двумя серверными событиями с одним `event_id` (в задании сказано только про дубли с пикселем) | derived-unverified | docs/task.md (только пиксель); аналогия с Meta CAPI в обоих исследованиях | open |
| A-004 | Суточный отчёт сети B содержит наш `event_id` либо `click_id` + тип + время | derived-unverified | docs/claude-deep-research.md §8 A-02; Gemini PDF ASSUMPTION-B1 | open |
| A-005 | Для B рекомендован вариант «только сервер» (пиксель Reg/FTD выключен); выбор между тремя вариантами за маркетингом | design-decision | README §7; docs/task.md (B не убирает дубли) | open |
| A-006 | Сеть C убирает дубли по `txid`/`order_id`, принимает обратный постбэк, а отмена неизвестного ей `txid` безвредна (отклоняется) | derived-unverified | docs/claude-deep-research.md §8 A-03, A-04; Gemini PDF ASSUMPTION-C1 | open |
| A-007 | C платит один FTD за пользователя; атрибуция Last Non-Direct Paid Click в окне 30 дней | derived-unverified | Gemini PDF C-03 | open |
| A-008 | В монолит можно добавить таблицу `conversion_outbox`, INSERT в транзакциях регистрации, FTD и чарджбэка и поле `email_sha256` в payload; сущность чарджбэка в монолите есть | design-decision | README §5.1; требует аппрува мейнтейнеров | open |
| A-009 | Фронт знает `user_id` и `deposit_id` и строит из них `event_id`; FTD-пиксель срабатывает после подтверждения пополнения, а не на попытке | design-decision | README §5.2 | open |
| A-010 | Все зеркала за нашим reverse proxy на одном IP-блоке; ссылки между ними мы можем декорировать | derived-unverified | Gemini PDF G-01; docs/claude-deep-research.md A-05 | open |
| A-011 | Safari ITP ограничивает 7 сутками cookie из JS (24 ч при декорированной ссылке) и Set-Cookie с чужого CNAME/IP | derived-unverified | Gemini PDF §4.4; ориентир https://webkit.org/tracking-prevention/ (не открыт, дата не проверена) | open |
| A-012 | Есть трафик EEA/UK. До решения юристов захват Click ID и cookie для EEA/UK идёт только после согласия (fail-closed); хэш email — персональные данные, нужен DPA | design-decision | Gemini PDF §6, G-02 | open |
| A-013 | Коэффициенты ×3 к пику, 3% чарджбэков, 10% click→registration и пример `t=5%`, `p=0,7` — иллюстративные | derived-unverified | Gemini PDF G-03, §5.1 | open |
| A-014 | Сеть C принимает постбэки только с allowlist IP и секретным токеном: нужны статические egress-IP | derived-unverified | Gemini PDF §3.3 п.5 | open |
| A-015 | Горизонт ретраев C — 7 суток; у партнёра есть locking period, после которого отмена идёт только спором | design-decision | Gemini PDF §5.2; Claude предлагал 72 ч | open |
| A-016 | MariaDB версии ≥ 10.6 (нужен `SKIP LOCKED`); иначе захват задачи через `UPDATE … WHERE status='PENDING'` с токеном аренды | derived-unverified | версия не указана в docs/task.md | open |
| A-017 | Неясный таймаут у C: ждём 1 ч и повторяем один раз; возможный дубль ловит месячная сверка (категория DUPLICATE_AT_PARTNER) | design-decision | README §5.4 | open |
| A-018 | Контракты API сетей A, B, C и API захвата в README §6 — предположения, а не документация | design-decision | README §6 | open |
| A-019 | Сверка B идёт по отчётам за D и D+1; досланные после неё события нарушают 15-минутный SLO и считаются отдельно | design-decision | README §5.4 | open |
| A-020 | Критичных уязвимостей сейчас нет: чувствительные данные и платёжная информация не скомпрометированы. Нужно подтвердить до отправки сообщения | user-stated 2026-11-20 | диалог с автором | open |
| A-021 | Около 20% мощности команды держим в резерве под критичную уязвимость | user-stated 2026-11-20 | диалог с автором | confirmed |
| A-022 | Приём запросов: один канал, карточка запроса (кто, что, дата, цена задержки, системы, размер, зависимости), ежедневный разбор, решение за лидом, очередь видна заказчикам | design-decision | README, часть 2, вопрос 1; поля карточки из `modes/intake.md` | open |
