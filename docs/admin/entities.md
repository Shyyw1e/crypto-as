# Сущности

## `user_entitlements`
Текущий доступ пользователя к боту (source of truth для подписки/trial).

### Fields
- `id` - PK
- `user_id` - FK -> `users.id`, unique
- `status` - `none | trial | active | expired | blocked`
- `trial_started_at` - timestamp, nullable
- `trial_ends_at` - timestamp, nullable
- `subscription_started_at` - timestamp, nullable
- `subscription_ends_at` - timestamp, nullable
- `trial_used` - bool, default `false`
- `blocked_reason` - text, nullable
- `created_at` - timestamp
- `updated_at` - timestamp

### Notes
- Одна запись на пользователя.
- Статус и сроки обновляются атомарно.
- Используется анализатором как главный гейт доступа.

## `admin_actions_log`
Аудит всех действий админов.

### Fields
- `id` - PK
- `admin_name` - text
- `target_user_id` - FK -> `users.id`
- `action_type` - `grant_trial | activate_sub | block | unblock | set_expired`
- `old_status` - text
- `new_status` - text
- `payload_json` - jsonb (причина, даты, комментарий)
- `created_at` - timestamp

### Notes
- Лог не редактируется и не удаляется.
- Любое изменение `user_entitlements` должно писать запись сюда.

## `payments` (future-ready)
Техническая таблица под платежные события.

### Fields
- `id` - PK
- `user_id` - FK -> `users.id`
- `provider` - text (`yookassa`, `cloudpayments`, ...)
- `provider_payment_id` - text, unique
- `amount` - numeric
- `currency` - text (например, `RUB`)
- `status` - `pending | succeeded | failed | refunded`
- `paid_at` - timestamp, nullable
- `created_at` - timestamp

### Notes
- На MVP можно не использовать в логике доступа.
- Нужна для связки webhook -> entitlement в следующем этапе.
