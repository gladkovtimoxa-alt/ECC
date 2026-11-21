---
description: Исправлять ошибки сборки C++, проблемы CMake и проблемы линковщика инкрементально. Вызывает агента cpp-build-resolver для минимальных, хирургических исправлений.
---

# C++ Build and Fix

Эта команда вызывает агента **cpp-build-resolver** для инкрементального исправления ошибок сборки C++ минимальными изменениями.

## Что делает эта команда

1. **Запустить диагностику**: выполнить `cmake --build`, `clang-tidy`, `cppcheck`
2. **Разобрать ошибки**: сгруппировать по файлу и отсортировать по серьёзности
3. **Инкрементально исправить**: по одной ошибке за раз
4. **Проверить каждое исправление**: перезапустить сборку после каждого изменения
5. **Отчитаться резюме**: показать, что исправлено и что осталось

## Когда использовать

Используйте `/cpp-build`, когда:
- `cmake --build build` падает с ошибками
- Ошибки линковщика (undefined references, multiple definitions)
- Сбои инстанцирования шаблонов
- Проблемы include/зависимостей
- После получения изменений, ломающих сборку

## Запускаемые диагностические команды

```bash
# Конфигурация CMake
cmake -B build -S .

# Сборка
cmake --build build 2>&1 | head -100

# Статический анализ (если доступен)
clang-tidy src/*.cpp -- -std=c++17
cppcheck --enable=all src/
```

## Пример сессии

```text
User: /cpp-build

Agent:
# C++ Build Resolution

## Начальная диагностика

```bash
$ cmake --build build
src/service/user.cpp:25:15: error: use of undeclared identifier 'UserRepository'
src/handler/api.cpp:42:9: error: no matching function for call to 'process'
src/handler/api.cpp:58:1: error: non-void function does not return a value
```

Найдено ошибок: 3

## Исправление 1: неопределённый идентификатор

Файл: src/service/user.cpp:25
Ошибка: use of undeclared identifier 'UserRepository'
Причина: отсутствующий include

```cpp
// Добавлен include
#include "repository/user_repository.hpp"
```

```bash
$ cmake --build build
# Осталось 2 ошибки
```

## Исправление 2: нет подходящей функции

Файл: src/handler/api.cpp:42
Ошибка: no matching function for call to 'process'

```cpp
// Было
process(params.get("count"));
// Стало
process(std::stoi(params.get("count")));
```

```bash
$ cmake --build build
# Осталась 1 ошибка
```

## Исправление 3: отсутствующий return

Файл: src/handler/api.cpp:58
Ошибка: non-void function does not return a value

```cpp
std::optional<User> getUser(const std::string& id) {
    if (id.empty()) {
        return std::nullopt;
    }
    auto user = findUser(id);
    // Добавлен отсутствующий return
    return user;
}
```

```bash
$ cmake --build build
# Сборка успешна!
```

## Финальная проверка

```bash
$ ctest --test-dir build --output-on-failure
Test project build
    1/5 Test #1: unit_tests ........   Passed    0.02 sec
    2/5 Test #2: integration_tests    Passed    0.15 sec
All tests passed.
```

## Резюме

| Метрика | Количество |
|--------|-------|
| Исправлено ошибок сборки | 3 |
| Исправлено ошибок линковщика | 0 |
| Изменено файлов | 2 |
| Оставшихся проблем | 0 |

Build Status: PASS: SUCCESS
```

## Распространённые исправленные ошибки

| Ошибка | Типичное исправление |
|-------|-------------|
| `undeclared identifier` | Добавить `#include` или исправить опечатку |
| `no matching function` | Исправить типы аргументов или добавить перегрузку |
| `undefined reference` | Подключить библиотеку или добавить реализацию |
| `multiple definition` | Использовать `inline` или перенести в .cpp |
| `incomplete type` | Заменить forward decl на `#include` |
| `no member named X` | Исправить имя члена или include |
| `cannot convert X to Y` | Добавить подходящее приведение |
| `CMake Error` | Исправить конфигурацию CMakeLists.txt |

## Стратегия исправления

1. **Сначала ошибки компиляции** — код должен компилироваться
2. **Затем ошибки линковщика** — разрешить undefined references
3. **Затем предупреждения** — исправлять с `-Wall -Wextra`
4. **Одно исправление за раз** — проверять каждое изменение
5. **Минимальные изменения** — не рефакторить, только исправлять

## Условия останова

Агент остановится и доложит, если:
- Та же ошибка сохраняется после 3 попыток
- Исправление вводит больше ошибок
- Требуются архитектурные изменения
- Отсутствующие внешние зависимости

## Связанные команды

- `/cpp-test` — запустить тесты после успешной сборки
- `/cpp-review` — проверить качество кода
- навык `verification-loop` — полный цикл верификации

## Связанное

- Агент: `agents/cpp-build-resolver.md`
- Навык: `skills/cpp-coding-standards/`
