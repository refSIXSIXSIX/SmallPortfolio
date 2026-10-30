# Finance Forecast System — очередь задач и повторные попытки (retry_comp)

Командный учебный проект МГТУ «СТАНКИН» (дисциплина «Информационные системы и технологии», сентябрь 2026 – н.в., команда из 5 человек).
Система анализа и прогнозирования финансовых показателей российских компаний с использованием ML (XGBoost, ARIMA).

> Командный репозиторий: https://github.com/Jjots/finance-forecast-app

## Моя зона ответственности

- **Архитектура (модель C4):** участвовал в L1–L2, отвечал за контейнеры `queue` (RabbitMQ) и `scheduler` на уровне компонентов (L3): компоненты, связи, сценарии публикации, доставки и повторной доставки задач на переобучение моделей.
- **Код (L4):** прототип компонента `retry_comp` — управление повторными попытками выполнения задач.

## retry_comp

Паттерн **Strategy**: политика задержки между попытками меняется во время работы, а `RetryExecutor` об этом не знает.

| Класс | Роль |
|-------|------|
| `RetryStrategy` (ABC) | контракт `execute(retry_number) -> float` — пауза перед повтором |
| `FixedDelayStrategy` | постоянная задержка (например, 2 с, 2 с, 2 с) |
| `ExponentialBackoffStrategy` | экспоненциальная задержка (2, 4, 8 с), опциональный потолок |
| `RetryExecutor` | контекст: выполняет задачу, меняет стратегию через `set_strategy` |
| `DeadLetterSink` / `InMemoryDeadLetterQueue` | приёмник окончательно неуспешных задач (аналог DLQ) |
| `RetryResult`, `RetryStatus`, `DeadLetter` | типизированные результаты: `SUCCESS`, `EXHAUSTED`, `PERMANENT_FAILURE` |

Поведение:
- временная ошибка → пауза по текущей стратегии → следующая попытка;
- после последней попытки паузы нет, задача уходит в DLQ со статусом `EXHAUSTED`;
- `PermanentError` прекращает повторы сразу.

## Качество

- 26 тестов `pytest` (положительные и отрицательные сценарии, ожидание подменяется имитацией);
- `mypy --strict` без ошибок;
- SOLID, строгая типизация, UML-диаграмма классов в PlantUML.

## Запуск

```bash
cd src/l4-code/danya-retry
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
PYTHONPATH=src python -m retry_comp.demo            # --real для настоящих пауз
PYTHONPATH=src python -m pytest -v
PYTHONPATH=src python -m mypy --strict src tests
```

## Ограничения

Это учебный прототип: RabbitMQ не запускался, TTL и dead-letter exchange не настраивались. В продакшене фиксированную паузу можно реализовать retry-очередью с TTL, а экспоненциальную — несколькими очередями или отложенной доставкой.
