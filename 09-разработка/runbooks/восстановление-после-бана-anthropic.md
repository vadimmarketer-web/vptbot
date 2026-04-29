---
тип: runbook
создан: 2026-04-29
owner: vadim
---

# Runbook: восстановление после бана Anthropic

## Контекст
Если Anthropic заблокирует основной аккаунт `vadim.marketer@gmail.com`, теряется доступ к Claude Code сессиям, истории чатов в claude.ai и API. Цель — за 1 час поднять рабочее окружение под новым аккаунтом.

## Что НЕ теряется (живёт вне Anthropic)
- Весь код в git (`vadimmarketer-web`, GitHub).
- Все vault'ы: `vptbot`, `phg`, `eurasia`.
- VPS `194.31.173.177` со всеми ботами и данными.
- Бэкапы в `/home/edgelab/backups/` на VPS.
- Зашифрованный архив секретов в Telegram (плюс пароль в пасс-менеджере).

## Что точно теряется
- История сессий и проектов в claude.ai (web).
- Подписка Claude (нужно купить заново на новом аккаунте).
- Дизайн-проекты в Claude Design (если есть).

## Шаги восстановления

### 1. Apple-приём (опционально, ~5 мин)
- Подать апелляцию: https://claude.ai/form/account-ban-appeals
- Шанс ~3.4% (по статистике Anthropic 2025: 50k апелляций → 1.7k отмен).
- Ждать ответа дольше 24 часов смысла нет — параллельно стартуем переезд.

### 2. Создание нового аккаунта (~10 мин)
- Регистрация на новом email (рабочем, не временном).
- Купить Claude Pro/Max **той же картой что и раньше** (долларовая, прямой биллинг).
- Никаких гифт-кодов, никаких индийских прокси-платежей.
- Зайти в Claude Code, чтобы сгенерировался новый `accountId`/`orgId`.

### 3. Восстановление конфига (~15 мин)
На VPS уже всё есть. Если терминал на VPS работает (а он работает, бан Anthropic ≠ потеря SSH):

```bash
ssh edgelab@194.31.173.177
ls /home/edgelab/backups/claude-config/   # последний tar.gz
```

Бэкап делается раз в 3 дня в 03:30, держим 30 копий.

Что внутри `claude-config-*.tar.gz`:
- `~/.claude/settings.json`
- `~/.claude/projects/<project>/memory/*` — auto-memory
- `~/.claude/projects/<project>/*.jsonl` — история сессий
- `~/.claude/agents/`, `skills/`, `plugins/`, `hooks/`

Восстановление на той же машине (если после бана всё стёрли):
```bash
cd /home/edgelab
tar xzf /home/edgelab/backups/claude-config/claude-config-<последний>.tar.gz
# ~/.claude перезапишется
```

### 4. Перенос сессий в Claude Code App (~10 мин)
Если используешь приложение Claude Code (macOS / Windows):

- Из бэкапа достать `~/.claude/projects/<project>/*.jsonl`.
- На macOS:
  ```
  ~/Library/Application Support/Claude/claude-code-sessions/<новый_accountId>/<новый_orgId>/
  ```
- Скопировать туда старые `local_*.json` / `*.jsonl`.
- В файлах `local-agent-mode-sessions/.../local_*.json` заменить `accountName` и `emailAddress` на значения из новой тестовой сессии.
- Перезапустить приложение.

### 5. Восстановление секретов (~5 мин)
На локальной машине (или VPS):
```bash
gpg -d secrets-2026-04-29.tar.gz.gpg > secrets.tar.gz
tar xzf secrets.tar.gz -C /home/edgelab/.claude-lab/shared/
```
Пароль — в пасс-менеджере под именем «Claude secrets backup».

Если нужно обновить — запустить вручную:
```bash
cd /home/edgelab/.claude-lab/shared && tar czf /tmp/secrets.tar.gz secrets/
gpg --symmetric --cipher-algo AES256 /tmp/secrets.tar.gz
mv /tmp/secrets.tar.gz.gpg /home/edgelab/backups/secrets/secrets-$(date +%Y-%m-%d).tar.gz.gpg
```

### 6. Восстановление данных ботов (если требуется)
Боты крутятся как `systemd`-юниты, их данные не теряются от бана Anthropic. Бэкап `/home/edgelab/backups/bot-data/` нужен только при сбое VPS:
```bash
tar xzf /home/edgelab/backups/bot-data/bot-data-<последний>.tar.gz -C /
# распаковка восстанавливает абсолютные пути:
# .claude-lab/shared/design-feed/seen.db
# .claude-lab/shared/gateway/state/*
# .claude-lab/shared/gateway/config.json
# .claude-lab/triss/workspace/*
# .claude-lab/triss/gateway/triss.env
```

### 7. API-ключ Anthropic (отдельно от подписки)
- Создать API-ключ на новом аккаунте: https://console.anthropic.com/
- Положить в `~/.claude-lab/shared/secrets/anthropic-api-key`
- Перезапустить ботов, использующих API (если такие есть).

## Профилактика (что делаем сейчас, чтобы потом не было больно)

| Что | Кадd | Где |
|-----|------|-----|
| Бэкап секретов (зашифрованный) | вручную при добавлении токена | `/home/edgelab/backups/secrets/`, копия в TG + пасс-менеджер |
| Бэкап `~/.claude/` | раз в 3 дня, авто | `/home/edgelab/backups/claude-config/`, держим 30 копий |
| Бэкап данных ботов | ежедневно, авто | `/home/edgelab/backups/bot-data/`, держим 30 копий |
| Логи cron | в `/home/edgelab/backups/logs/` | проверять раз в неделю |
| Гигиена биллинга | всегда | прямая карта, никаких гифт-кодов |
| Один аккаунт на email | всегда | не плодить тестовые |

## Cron-задачи (на VPS)
```cron
# bot-data: ежедневно 03:00
0 3 * * *     /home/edgelab/backups/scripts/backup.sh bot-data >> /home/edgelab/backups/logs/cron.log 2>&1
# claude-config: раз в 3 дня, 03:30
30 3 */3 * *  /home/edgelab/backups/scripts/backup.sh claude-config >> /home/edgelab/backups/logs/cron.log 2>&1
```

## Проверка работоспособности (раз в неделю, 1 минута)
```bash
ls -lt /home/edgelab/backups/claude-config/ | head -3
ls -lt /home/edgelab/backups/bot-data/ | head -3
tail -10 /home/edgelab/backups/logs/backup.log
```

Если последний архив старше расписания + 24 часа — что-то сломалось, разбираемся.
