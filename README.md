# Forecast Multi-Agent — многоагентный прогнозный pipeline на OpenRouter

Учебный проект по курсу «Генеративные задачи в NLP II».  
Реализует многоагентную систему, в которой несколько больших языковых моделей (LLM) совместно отвечают на аналитически‑прогнозный запрос пользователя.

---

## Описание проекта

Пользователь формулирует запрос — например, «Спрогнозируй проникновение ИИ-репетиторов в школьное образование (K-12) на горизонте 7–10 лет». Система автоматически:

1. **Анализирует сложность** задачи и составляет план через **Orchestrator** (LLM-агент).
2. **Подбирает ансамбль** из 2–10 LLM-агентов (`WorkerPool`), запускает их параллельно через OpenRouter.
3. **Оценивает** каждый ответ через **Evaluator** (LLM-судья, порог релевантности 0.55).
4. **Агрегирует** релевантные ответы в структурированный итоговый прогноз через **Summarizer**.

Весь ход работы фиксируется: каждый LLM-запрос, ответ, оценка и итог сохраняются в файлы трассировки.

```

| Компонент | Класс | Роль |
|-----------|-------|------|
| Транспорт | `OpenRouterClient` | Асинхронные запросы к OpenRouter, retry через `tenacity`, учёт токенов и стоимости |
| Кеш | `ResponseCache` | Файловый SHA-1 кеш по `(model, prompt, params)` |
| Логирование | `Tracer` | Запись событий pipeline в JSONL и читаемый `.log` |
| Планировщик | `Orchestrator` | Оценка сложности, подбор моделей, генерация ideal_prompt |
| Исполнители | `WorkerAgent` / `WorkerPool` | Параллельный запуск через `asyncio.gather` |
| Контроль качества | `Evaluator` | LLM-судья, порог `RELEVANCE_THRESHOLD = 0.55` |
| Агрегатор | `Summarizer` | Синтез консенсуса / прогнозов / допущений / рисков |

### Доступные LLM-агенты (демо)

| Модель | Провайдер | Роль в ансамбле |
|--------|-----------|-----------------|
| `mistralai/mistral-large-2411` | Mistral AI | balanced, json-output |
| `mistralai/mixtral-8x22b-instruct` | Mistral AI | MoE, multilingual |
| `meta-llama/llama-3.3-70b-instruct` | Meta | fast, longform |
| `deepseek/deepseek-r1` | DeepSeek | reasoning, deliberation |
| `deepseek/deepseek-chat` | DeepSeek | general, fast |
| `qwen/qwen3-235b-a22b` | Alibaba | MoE, multilingual, reasoning |
| `moonshotai/kimi-k2-0905` | Moonshot AI | long-context, agentic |
| `deepseek/deepseek-v3.1-terminus` | DeepSeek | reasoning, fast |
| `nousresearch/hermes-4-70b` | Nous Research | instruction-following, structured-output |
| `cohere/command-r-plus-08-2024` | Cohere | RAG, multilingual, longform |

> Все модели доступны через единый OpenRouter API. Модели OpenAI / Anthropic / Google **не используются** из-за ограничений тестового ключа.

```

## Файлы трассировки (логи)

При каждом запуске pipeline автоматически создаются два файла в текущей директории (или в `artifacts/traces/`, если папка существует):

```
run-<YYYYMMDD-HHmmss>.jsonl
run-<YYYYMMDD-HHmmss>.log
```

В репозитории приложен результат реального демо-запуска от **22 апреля 2026 г.** (запрос о проникновении ИИ-репетиторов в K-12 образование на горизонте 7–10 лет):

| Файл | Размер | Строк | Формат |
|------|--------|-------|--------|
| `run-20260422-144334.jsonl` | 150 КБ | 38 | JSON Lines — по одной записи на событие |
| `run-20260422-144334.log` | 154 КБ | 1 965 | Читаемый текстовый лог с временны́ми метками |

### JSONL-трассировка (`*.jsonl`)

Каждая строка — самостоятельный JSON-объект с обязательными полями `ts` (Unix timestamp), `run_id` и `event`.

**Типы событий и их поля:**

| `event` | Когда записывается | Ключевые поля |
|---------|-------------------|---------------|
| `user_query` | Начало запуска | `query` |
| `llm_request` | Перед каждым обращением к LLM | `role` (`orchestrator` / `worker:<model>` / `evaluator:<model>` / `summarizer`), `model`, `messages`, `params` |
| `llm_response` | После получения ответа LLM | `role`, `model`, `content`, `latency_s`, `cost_usd`, `tokens`, `cached`, `error` |
| `cache_hit` | Если ответ взят из кеша | `model`, `key` |
| `llm_error` | При ошибке API | `model`, `error`, `attempt` |
| `planned` | После формирования плана Orchestrator'ом | `complexity`, `models`, `rationale`, `ideal_prompt` |
| `plan_confirmed` | После принятия плана пользователем | `complexity`, `models`, `ideal_prompt` |
| `workers_done` | После завершения всех Worker-агентов | `n_total`, `n_ok`, `n_error` |
| `eval_done` | После оценки всех ответов Evaluator'ом | `n_relevant`, `scores` (dict model→score) |
| `summary_done` | После генерации итогового прогноза | `length`, `final_forecast` |

**Пример записи `llm_response`:**
```json
{
  "ts": 1776869032.848,
  "run_id": "run-20260422-144334",
  "event": "llm_response",
  "role": "orchestrator",
  "model": "mistralai/mistral-large-2411",
  "content": "{ ... }",
  "latency_s": 18.3,
  "cost_usd": 0.002782,
  "tokens": 1391,
  "cached": false,
  "error": null
}
```

JSONL удобен для программного анализа:

```python
import json, pandas as pd

events = [json.loads(l) for l in open("run-20260422-144334.jsonl", encoding="utf-8")]
responses = [e for e in events if e["event"] == "llm_response"]
df = pd.DataFrame(responses)
print(df[["role", "model", "latency_s", "cost_usd", "tokens"]].to_string())
```

### Читаемый лог (`*.log`)

Текстовый файл в формате:

```
=== RUN run-20260422-144334 STARTED @ 2026-04-22 14:43:34 ===
14:43:34 | INFO    | [1/5] Orchestrator: планирование:
--- REQUEST  orchestrator / mistralai/mistral-large-2411 ---
[system] ...системный промпт...
[user]   ...запрос пользователя...
--- RESPONSE  18.30s  $0.002782  1391 tok ---
...ответ модели...
========================================================================
14:43:52 | INFO    | [2/5] WorkerPool: запуск 7 агентов параллельно
...
```

Лог содержит:
- **Заголовок** каждой стадии (`[1/5] Orchestrator`, `[2/5] WorkerPool`, `[3/5] Evaluator`, `[4/5] Summarizer`, `[5/5] Done`).
- **Полные тексты** системных и пользовательских промптов для каждого вызова LLM.
- **Полные тексты ответов** всех Worker-агентов, оценок Evaluator'а и итогового прогноза Summarizer'а.
- **Метрики** каждого вызова: задержка (сек.), стоимость (USD), количество токенов.
- **Сводку** по запуску в конце: список релевантных ответов, итоговый прогноз.

### Результаты демо-запуска (22.04.2026)

**Запрос:** прогноз проникновения ИИ-репетиторов в школьное образование (K-12) на горизонте 7–10 лет.

| Параметр | Значение |
|----------|---------|
| Сложность (Orchestrator) | 8 / 10 |
| Подобрано агентов | 7 |
| Релевантных ответов | 7 / 7 (100 %) |
| Модель Orchestrator | `mistralai/mistral-large-2411` |
| Модель Evaluator | `mistralai/mistral-large-2411` |
| Модель Summarizer | `meta-llama/llama-3.3-70b-instruct` |
| Стоимость Orchestrator | $0.002782 |
| Стоимость Summarizer | $0.003605 |

---

## Конфигурация

Основные параметры задаются в первой исполняемой ячейке ноутбука:

| Параметр | Значение по умолчанию | Описание |
|----------|-----------------------|----------|
| `ORCHESTRATOR_MODEL` | `mistralai/mistral-large-2411` | Модель для планирования |
| `EVALUATOR_MODEL` | `mistralai/mistral-large-2411` | Модель-судья |
| `SUMMARIZER_MODEL` | `meta-llama/llama-3.3-70b-instruct` | Модель финальной свёртки |
| `RELEVANCE_THRESHOLD` | `0.55` | Минимальный score для включения в свёртку |
| `MAX_CONCURRENT` | `6` | Лимит параллельных запросов к OpenRouter |
| `AVAILABLE_MODELS` | 10 моделей | Список, из которого Orchestrator выбирает ансамбль |

---

## Лицензия

Проект учебный. Использование разрешено.
