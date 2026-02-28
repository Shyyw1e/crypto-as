# Access Status Model

## Statuses
- `none` - у пользователя еще не было доступа (trial/подписки).
- `trial` - активен пробный период.
- `active` - активна оплаченная или вручную выданная подписка.
- `expired` - доступ закончился.
- `blocked` - доступ принудительно заблокирован админом.

## Transitions
- `none -> trial` (`start_trial`)
- `none -> active` (`activate_subscription`, `payment_succeeded`)
- `trial -> active` (`payment_succeeded`, `activate_subscription`)
- `trial -> expired` (`trial_expired`)
- `active -> expired` (`subscription_expired`, `manual_set_expired`)
- `any -> blocked` (`admin_block`)
- `blocked -> expired` (`admin_unblock_without_extension`)
- `blocked -> active` (`admin_unblock_with_extension`, `admin_activate`)

## Validation Rules
- Если статус `blocked`, доступ всегда запрещен независимо от сроков.
- Если `subscription_ends_at` в прошлом и статус не `blocked`, статус должен стать `expired`.
- Если `trial_ends_at` в прошлом и статус `trial`, статус должен стать `expired`.
- Для `trial` обязательно `trial_started_at`, `trial_ends_at`.
- Для `active` обязательно `subscription_started_at`, `subscription_ends_at`.
- `trial_used=true` после первого старта trial и никогда не сбрасывается автоматически.

## Access Check Priority
1. Проверить `blocked`.
2. Проверить валидность сроков (`trial_ends_at` / `subscription_ends_at`).
3. Разрешить доступ только для `trial` или `active` с актуальными сроками.
