---
description: Применять рабочий процесс TDD для C++. Сначала писать тесты GoogleTest, затем реализовывать. Проверять покрытие через gcov/lcov.
---

# C++ TDD Command

Эта команда применяет методологию test-driven development для кода C++ с использованием GoogleTest/GoogleMock с CMake/CTest.

## Что делает эта команда

1. **Определить интерфейсы**: сначала скаффолдить сигнатуры класса/функции
2. **Написать тесты**: создать комплексные тест-кейсы GoogleTest (RED)
3. **Запустить тесты**: проверить, что тесты падают по правильной причине
4. **Реализовать код**: написать минимальный код, чтобы пройти (GREEN)
5. **Рефакторинг**: улучшить, сохраняя тесты зелёными
6. **Проверить покрытие**: обеспечить 80%+ покрытие

## Когда использовать

Используйте `/cpp-test`, когда:
- Реализуете новые функции или классы C++
- Добавляете покрытие тестами к существующему коду
- Исправляете баги (сначала пишете падающий тест)
- Строите критичную бизнес-логику
- Изучаете рабочий процесс TDD в C++

## Цикл TDD

```
RED     → Написать падающий тест GoogleTest
GREEN   → Реализовать минимальный код, чтобы пройти
REFACTOR → Улучшить код, тесты остаются зелёными
REPEAT  → Следующий тест-кейс
```

## Пример сессии

````
User: /cpp-test нужна функция для валидации email-адресов

Agent:
# TDD-сессия: валидатор email

## Шаг 1: определить интерфейс

```cpp
// validator/email.hpp
#pragma once
#include <string>
#include <expected>

enum class EmailError {
    Empty,
    InvalidFormat
};

std::expected<void, EmailError> validate_email(const std::string& email);
```

## Шаг 2: написать тесты (RED)

```cpp
// validator/email_test.cpp
#include <gtest/gtest.h>
#include "email.hpp"

TEST(ValidateEmail, AcceptsSimpleEmail) {
    auto result = validate_email("user@example.com");
    EXPECT_TRUE(result.has_value());
}

TEST(ValidateEmail, AcceptsSubdomain) {
    EXPECT_TRUE(validate_email("user@mail.example.com").has_value());
}

TEST(ValidateEmail, AcceptsPlus) {
    EXPECT_TRUE(validate_email("user+tag@example.com").has_value());
}

TEST(ValidateEmail, RejectsEmpty) {
    auto result = validate_email("");
    ASSERT_FALSE(result.has_value());
    EXPECT_EQ(result.error(), EmailError::Empty);
}

TEST(ValidateEmail, RejectsNoAtSign) {
    EXPECT_FALSE(validate_email("userexample.com").has_value());
}

TEST(ValidateEmail, RejectsNoDomain) {
    EXPECT_FALSE(validate_email("user@").has_value());
}

TEST(ValidateEmail, RejectsNoLocalPart) {
    EXPECT_FALSE(validate_email("@example.com").has_value());
}
```

## Шаг 3: запустить тесты — проверить FAIL

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....***Failed
    --- undefined reference to `validate_email`

FAIL
```

✓ Тесты падают как ожидалось (не реализовано).

## Шаг 4: реализовать минимальный код (GREEN)

```cpp
// validator/email.cpp
#include "email.hpp"
#include <regex>

std::expected<void, EmailError> validate_email(const std::string& email) {
    if (email.empty()) {
        return std::unexpected(EmailError::Empty);
    }
    static const std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
    if (!std::regex_match(email, pattern)) {
        return std::unexpected(EmailError::InvalidFormat);
    }
    return {};
}
```

## Шаг 5: запустить тесты — проверить PASS

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....   Passed    0.01 sec

100% tests passed.
```

✓ Все тесты проходят!

## Шаг 6: проверить покрытие

```bash
$ cmake -DCMAKE_CXX_FLAGS="--coverage" -B build && cmake --build build
$ ctest --test-dir build
$ lcov --capture --directory build --output-file coverage.info
$ lcov --list coverage.info

validator/email.cpp     | 100%
```

✓ Покрытие: 100%

## TDD завершён!
````

## Паттерны тестов

### Базовые тесты
```cpp
TEST(SuiteName, TestName) {
    EXPECT_EQ(add(2, 3), 5);
    EXPECT_NE(result, nullptr);
    EXPECT_TRUE(is_valid);
    EXPECT_THROW(func(), std::invalid_argument);
}
```

### Фикстуры
```cpp
class DatabaseTest : public ::testing::Test {
protected:
    void SetUp() override { db_ = create_test_db(); }
    void TearDown() override { db_.reset(); }
    std::unique_ptr<Database> db_;
};

TEST_F(DatabaseTest, InsertsRecord) {
    db_->insert("key", "value");
    EXPECT_EQ(db_->get("key"), "value");
}
```

### Параметризованные тесты
```cpp
class PrimeTest : public ::testing::TestWithParam<std::pair<int, bool>> {};

TEST_P(PrimeTest, ChecksPrimality) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(is_prime(input), expected);
}

INSTANTIATE_TEST_SUITE_P(Primes, PrimeTest, ::testing::Values(
    std::make_pair(2, true),
    std::make_pair(4, false),
    std::make_pair(7, true)
));
```

## Команды покрытия

```bash
# Сборка с покрытием
cmake -DCMAKE_CXX_FLAGS="--coverage" -DCMAKE_EXE_LINKER_FLAGS="--coverage" -B build

# Запуск тестов
cmake --build build && ctest --test-dir build

# Генерация отчёта покрытия
lcov --capture --directory build --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_html
```

## Цели покрытия

| Тип кода | Цель |
|-----------|--------|
| Критичная бизнес-логика | 100% |
| Публичные API | 90%+ |
| Общий код | 80%+ |
| Сгенерированный код | Исключить |

## Лучшие практики TDD

**ДЕЛАТЬ:**
- Писать тест СНАЧАЛА, до любой реализации
- Запускать тесты после каждого изменения
- Использовать `EXPECT_*` (продолжает) вместо `ASSERT_*` (останавливает), когда уместно
- Тестировать поведение, не детали реализации
- Включать граничные случаи (пусто, null, максимальные значения, граничные условия)

**НЕ ДЕЛАТЬ:**
- Писать реализацию до тестов
- Пропускать фазу RED
- Тестировать приватные методы напрямую (тестировать через публичный API)
- Использовать `sleep` в тестах
- Игнорировать нестабильные тесты

## Связанные команды

- `/cpp-build` — исправить ошибки сборки
- `/cpp-review` — проверить код после реализации
- навык `verification-loop` — запустить полный цикл верификации

## Связанное

- Навык: `skills/cpp-testing/`
- Навык: `skills/tdd-workflow/`
