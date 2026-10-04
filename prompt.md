# ЗАДАЧА: Выполнить Лабораторную работу № 2 «Проектирование архитектуры программной AI-системы»

Ты — архитектурный ассистент. Твоя задача — создать **только документацию и структуру репозитория** для проектируемой AI-системы. Код приложения (FastAPI, RAG, агент, Gradio) писать НЕ нужно — он будет реализован позже отдельно.

После выполнения — закоммить и запушить.

## Контекст проекта

**Тема:** Внутренний AI-ассистент для команды Data Science по документации scikit-learn.

**Что за сервис:**
- RAG-сервис над документацией scikit-learn (FastAPI + Qdrant + Gradio).
  - Корпус: 3 модуля scikit-learn (linear_model, tree, model_evaluation) + about.md → ~267 чанков.
  - Эмбеддер: `intfloat/multilingual-e5-small` (384-dim, мультиязычный).
  - LLM: через OpenRouter (OpenAI-совместимый API), модель `meta-llama/llama-3.3-70b-instruct`.
  - API: `POST /chat` (RAG), `GET /health`, `GET /metrics`.
  - UI: Gradio-чат с панелью таймингов и источников.
- Расширение LangGraph-агентом.
  - 3 инструмента: `documentation_search` (обёртка над RAG), `python_repl` (PythonREPLTool), `web_search` (DuckDuckGo).
  - ReAct-цикл: agent_node → tool_executor → agent_node.
  - Лимит итераций: 5. Checkpointing: MemorySaver. Guardrails на вход/выход.
  - API: `POST /agent` с полем `trace` (пошаговый журнал вызовов инструментов).
  - UI: переключатель режимов «Быстрый (/chat)» / «Агент (/agent)», статус-строка, блок «Что сделал агент».

**Стек:** FastAPI, Pydantic v2, LangChain, LangGraph, Qdrant, sentence-transformers, Gradio, structlog, RAGAS, Docker, GitHub Actions, pytest.

---

## ТРЕБОВАНИЯ ЛР № 2 (обязательно выполнить все)

### Задача 1. Выбрать прикладной индустриальный кейс и сформулировать бизнес-проблему

- Трек № 5 «Enterprise RAG» + элементы трека № 4 «Smart Helpdesk».
- Бизнес-цель: сократить время поиска информации по ML с 10–15 минут до 5–10 секунд.
- Конечные потребители: Data Scientist, аналитик, руководитель.

### Задача 2. Идентифицировать пользователей, внешние системы и Use Cases

- Пользователи: Data Scientist, аналитик.
- Внешние системы: scikit-learn.org, Qdrant, OpenRouter, DuckDuckGo.
- Use Cases: задать вопрос по документации, выполнить вычисление, найти свежую информацию.

### Задача 3. Определить класс задач AI и целевые метрики

- Класс: RAG (Retrieval-Augmented Generation) + Agentic RAG (ReAct).
- Метрики модели: Recall@4 ≥ 0.80, Faithfulness ≥ 0.75, Response Relevancy ≥ 0.75, tool-choice accuracy.
- Метрики SLA: `/chat` p95 < 7 сек, `/agent` p95 < 20 сек, TTFT < 1 сек, доступность 99.9%.

### Задача 4. Выбрать и обосновать архитектурный стиль

- Модульный монолит в едином Docker-контейнере (app) + отдельный контейнер Qdrant.
- Альтернативы: микросервисы, event-driven. Обосновать отказ.

### Задача 5. Компонентная декомпозиция

Модули:
- `app/api/` — Pydantic-схемы, роуты
- `app/core/` — конфигурация, логирование
- `app/data/` — загрузка корпуса, чанкинг
- `app/ml/` — эмбеддер, LLM-клиент
- `app/services/` — бизнес-логика (RAG-цепочка, агент)
- `app/repositories/` — персистентность (БД, Qdrant)
- `app/rag/` — LCEL-цепочка
- `app/agent/` — tools, graph, prompts, guardrails
- `app/schemas/` — Pydantic-схемы запросов/ответов
- `app/scripts/` — утилиты (load_corpus, index_corpus)

### Задача 6. Контекстная и компонентная диаграммы (Mermaid)

- **C4 Level 1: System Context** — `flowchart LR` с центральным блоком «AI-Ассистент» и внешними системами (пользователь, scikit-learn.org, Qdrant, OpenRouter, DuckDuckGo, Monitoring, RAGAS).
- **C4 Level 2: Container / Component** — `flowchart TB` с подграфами AppContainer, ArtifactStore, DataStore, ObservabilityStack, модулями RAG и Agent.

### Задача 7. Спецификация REST API (Pydantic-схемы)

Эндпоинты:
- `POST /chat` — ChatRequest, ChatResponse, Source
- `POST /agent` — AgentRequest, AgentResponse, TraceStep, Source
- `GET /health` — `{"status": "ok"}`
- `GET /metrics` — OpenMetrics (упомянуть)

Для каждого — Pydantic-схема + пример JSON-запроса и ответа. Ошибки 422 — пример.

### Задача 8. Сквозные потоки данных (Data Flows)

4 диаграммы `sequenceDiagram` в Mermaid:
1. Индексация корпуса (offline)
2. RAG-запрос (online)
3. Agentic RAG (online, ReAct-цикл)
4. RAGAS-оценка

Для каждого — протокол, формат, задержки.

### Задача 9. Architecture Decision Records (ADR)

Ровно 3 ADR в `docs/adr/`, по шаблону: Статус, Контекст, Альтернативы (≥2), Решение, Обоснование, Последствия.

- **ADR-01:** Выбор режима инференса (Synchronous REST API vs Asynchronous Message Queue). Решение: синхронный REST API.
- **ADR-02:** Стратегия хранения и версионирования моделей. Решение: запрет на бинарные веса в Git; HuggingFace Cache + JSONL для чанков; в production — DVC + S3 или MLflow.
- **ADR-03:** Архитектурный стиль (Модульный монолит vs Микросервисы). Решение: модульный монолит.

### Задача 10. Безопасность, наблюдаемость, защита от дрейфа

- Guardrails: prompt injection patterns (regex), ограничение длины, allowed chars.
- Structured logging (structlog JSON) с полями: timestamp, level, event, thread_id, tool, latency_ms, iteration.
- Метрики Prometheus: http_requests_total, http_request_duration_seconds, model_inference_duration_seconds.
- RAGAS-метрики на golden dataset (10 вопросов): Recall@4, Faithfulness, Response Relevancy.
- Контроль дрейфа: Evidently, PSI, тест Колмогорова-Смирнова.
- 152-ФЗ: корпус — публичная документация; вопросы уходят в OpenRouter; в production — локальная LLM.

### Задача 11. Инициализация Git-репозитория на GitHub

- Структура каталогов (см. ниже).
- `.gitignore` с исключениями: `__pycache__/`, `.venv/`, `.env`, `*.pkl`, `*.pt`, `*.onnx`, `*.bin`, `*.safetensors`, `qdrant_data/`, `data/corpus_chunks.jsonl`, `.ipynb_checkpoints/`.
- `README.md` — полный архитектурный отчёт (разделы 1–10).
- Отдельные файлы: `docs/adr/ADR-01.md`, `ADR-02.md`, `ADR-03.md`, `docs/architecture.md`, `docs/api-contracts.md`.
- Коммиты (2 штуки):
  ```
  git add README.md docs/ .gitignore
  git commit -m "docs: add architecture specification for lab 2 (README + ADR + diagrams)"

  git add app/ data/ notebooks/ tests/
  git commit -m "chore: initialize project structure for lab 2"

  git push origin main
  ```

---

## Структура репозитория, которую нужно создать

```
ai-system-rag-assistant/
├── .gitignore
├── LICENSE                    # MIT (если ещё нет)
├── README.md                  # Полный архитектурный отчёт
├── docs/
│   ├── architecture.md        # Краткая выжимка + оглавление
│   ├── api-contracts.md       # Спецификация API отдельно
│   └── adr/
│       ├── ADR-01.md          # Режим инференса
│       ├── ADR-02.md          # Хранение моделей
│       └── ADR-03.md          # Архитектурный стиль
├── app/
│   ├── __init__.py
│   ├── api/__init__.py
│   ├── core/__init__.py
│   ├── data/__init__.py
│   ├── ml/__init__.py
│   ├── services/__init__.py
│   ├── repositories/__init__.py
│   ├── rag/__init__.py
│   ├── agent/__init__.py
│   ├── schemas/__init__.py
│   └── scripts/__init__.py
├── data/
│   └── local/
│       └── .gitkeep
├── notebooks/
│   └── .gitkeep
└── tests/
    └── __init__.py
```

**Важно:** все `__init__.py` — пустые файлы. Никакого кода внутри `app/` не создавай. Реализация появится позже.

---

## Что НЕ делать (критично)

- ❌ НЕ пиши код приложения: `main.py`, `config.py`, `llm.py`, `chain.py`, `tools.py`, `graph.py`, `guardrails.py`, `prompts.py`, `schemas/*.py` с реальной логикой.
- ❌ НЕ создавай `Dockerfile`, `docker-compose.yml`, `requirements.txt`.
- ❌ НЕ создавай `.github/workflows/`.
- ❌ НЕ коммить `.env`, веса моделей, `qdrant_data/`, `corpus_chunks.jsonl`.
- ❌ НЕ пиши тесты (`tests/test_*.py`) — только `tests/__init__.py`.
- ❌ НЕ создавай notebooks с кодом — только пустые `.gitkeep` в `notebooks/`.
- ❌ **НЕ упоминай в документации никакие курсы, уроки, недели, шаги или учебные программы.** Проект описывается как самостоятельный: «выбранный кейс», «проектируемая система», «в рамках лабораторной работы».

Если сомневаешься, писать ли файл — спроси у пользователя, а не пиши.

---

## Требования к оформлению

- Все диаграммы — **Mermaid** (flowchart, sequenceDiagram). Синтаксис должен быть корректным.
- Pydantic-схемы — с аннотациями типов и `Field(...)`.
- ADR — строго по шаблону (Статус, Контекст, Альтернативы, Решение, Обоснование, Последствия).
- README.md — разделы 1–10 (нумерация как в ЛР № 2).
- Язык — русский (термины и код — на английском).
- В README — таблица требований FR-01…FR-05 и NFR-01…NFR-04.
- В README — таблица компонентов (компонент / назначение / вход / выход / библиотеки).
- В README — таблица информационных потоков (поток / протокол / частота / формат).
- **Никаких отсылок к курсам, урокам, неделям, шагам, учебным программам.**

---

## Порядок работы

1. Создай `README.md` со всеми разделами 1–10.
2. Создай `docs/adr/ADR-01.md`, `ADR-02.md`, `ADR-03.md`.
3. Создай `docs/architecture.md` и `docs/api-contracts.md`.
4. Создай структуру каталогов с `__init__.py` и `.gitkeep`.
5. Обнови `.gitignore`.
6. Сделай 2 коммита и push.

После каждого крупного шага показывай пользователю `git diff --stat` и спрашивай, продолжать ли.

---

## Чек-лист перед завершением

- [ ] README содержит все 10 разделов
- [ ] В README есть контекстная и компонентная диаграммы Mermaid
- [ ] В README есть 4 sequenceDiagram для потоков данных
- [ ] В README есть таблицы: требования FR/NFR, компоненты, информационные потоки
- [ ] В README есть Pydantic-схемы для `/chat` и `/agent` с примерами JSON
- [ ] Созданы ровно 3 ADR (ADR-01, ADR-02, ADR-03) по шаблону с 6 полями
- [ ] В `.gitignore` есть `.env`, `qdrant_data/`, `*.pkl`, `*.pt`, `*.onnx`, `*.bin`, `*.safetensors`, `data/corpus_chunks.jsonl`
- [ ] Структура каталогов создана, все `__init__.py` пустые
- [ ] Нет кода приложения (только документация и структура)
- [ ] Нет `Dockerfile`, `docker-compose.yml`, `requirements.txt`, `.github/workflows/`
- [ ] Нет упоминаний курсов, уроков, недель, шагов, учебных программ
- [ ] Сделано 2 коммита с осмысленными сообщениями
- [ ] Push в `main` выполнен
- [ ] Mermaid-диаграммы визуально проверены на https://mermaid.live

Начинай с README.md. Если что-то непонятно — спрашивай, не додумывай.