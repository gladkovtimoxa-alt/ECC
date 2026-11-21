---
description: Создать, проверить или вывести список чекпоинтов рабочего процесса после запуска проверок верификации.
---

# Команда Checkpoint

Создать или проверить чекпоинт в вашем рабочем процессе.

## Использование

`/checkpoint [create|verify|list] [name]`

## Создание чекпоинта

При создании чекпоинта:

1. Запустите `/verify quick`, чтобы убедиться, что текущее состояние чисто
2. Создайте git stash или коммит с именем чекпоинта
3. Залогируйте чекпоинт в `.claude/checkpoints.log`:

```bash
echo "$(date +%Y-%m-%d-%H:%M) | $CHECKPOINT_NAME | $(git rev-parse --short HEAD)" >> .claude/checkpoints.log
```

4. Сообщите, что чекпоинт создан

## Проверка чекпоинта

При проверке против чекпоинта:

1. Прочитайте чекпоинт из лога
2. Сравните текущее состояние с чекпоинтом:
   - Файлы, добавленные с момента чекпоинта
   - Файлы, изменённые с момента чекпоинта
   - Процент прохождения тестов сейчас vs тогда
   - Покрытие сейчас vs тогда

3. Отчитайтесь:
```
CHECKPOINT COMPARISON: $NAME
============================
Files changed: X
Tests: +Y passed / -Z failed
Coverage: +X% / -Y%
Build: [PASS/FAIL]
```

## Список чекпоинтов

Покажите все чекпоинты с:
- Именем
- Временной меткой
- Git SHA
- Статусом (current, behind, ahead)

## Рабочий процесс

Типичный поток чекпоинтов:

```
[Start] --> /checkpoint create "feature-start"
   |
[Implement] --> /checkpoint create "core-done"
   |
[Test] --> /checkpoint verify "core-done"
   |
[Refactor] --> /checkpoint create "refactor-done"
   |
[PR] --> /checkpoint verify "feature-start"
```

## Аргументы

$ARGUMENTS:
- `create <name>` — создать именованный чекпоинт
- `verify <name>` — проверить против именованного чекпоинта
- `list` — показать все чекпоинты
- `clear` — удалить старые чекпоинты (оставляет последние 5)
