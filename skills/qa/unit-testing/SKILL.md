# Unit Testing

## Когда применять

Используй для тестирования отдельных функций,
методов и компонентов в изоляции.

## Workflow

1. Найди тестируемую логику.
2. Определи happy path.
3. Определи edge cases.
4. Определи error cases.
5. Создай тесты.
6. Запусти их.
7. Проверь coverage.

## Правила

- Не тестируй implementation details.
- Один тест должен проверять одну концепцию.
- Используй table-driven tests для похожих сценариев.
- Mock только внешние зависимости.
- Не используй реальные БД/API в unit-тестах.

## Структура

Given → When → Then

## Checklist

- [ ] happy path
- [ ] errors
- [ ] edge cases
- [ ] dependencies mocked
- [ ] tests pass
