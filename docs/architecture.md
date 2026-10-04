# Архитектура AI-Ассистента (краткая выжимка)

Полный архитектурный отчёт — в [README.md](../README.md).

## Оглавление

1. [Обзор](#обзор)
2. [Компоненты](#компоненты)
3. [Диаграммы](#диаграммы)
4. [Потоки данных](#потоки-данных)
5. [ADR](#adr)

## Обзор

Внутренний AI-ассистент для команды Data Science по документации scikit-learn. Состоит из двух режимов:

- **RAG** (`POST /chat`) — семантический поиск + генерация ответа.
- **Agentic RAG** (`POST /agent`) — ReAct-агент с инструментами `documentation_search`, `python_repl`, `web_search`.

Стек: FastAPI, Pydantic v2, LangChain, LangGraph, Qdrant, sentence-transformers, Gradio, structlog, RAGAS, Docker, GitHub Actions, pytest.

## Компоненты

| Модуль | Назначение |
|--------|-----------|
| `app/api/` | Pydantic-схемы, роуты |
| `app/core/` | Конфигурация, логирование |
| `app/data/` | Загрузка корпуса, чанкинг |
| `app/ml/` | Эмбеддер, LLM-клиент |
| `app/services/` | Бизнес-логика (RAG, агент) |
| `app/repositories/` | Персистентность (БД, Qdrant) |
| `app/rag/` | LCEL-цепочка |
| `app/agent/` | Tools, graph, prompts, guardrails |
| `app/schemas/` | Pydantic-схемы |
| `app/scripts/` | Утилиты (load_corpus, index_corpus) |

## Диаграммы

- **C4 Level 1 (System Context)** — см. README, раздел 6.
- **C4 Level 2 (Container / Component)** — см. README, раздел 6.
- **Sequence-диаграммы** (4 шт.) — см. README, раздел 8.

## Потоки данных

| Поток | Протокол | Частота | Формат |
|-------|----------|---------|--------|
| Индексация корпуса | HTTPS / gRPC | Единоразово | HTML → JSONL → vectors |
| RAG-запрос | HTTPS | On-demand | JSON |
| Agentic RAG | HTTPS | On-demand | JSON |
| RAGAS-оценка | Локальный | По расписанию | JSON |

## ADR

- [ADR-01: Режим инференса](adr/ADR-01.md)
- [ADR-02: Хранение и версионирование моделей](adr/ADR-02.md)
- [ADR-03: Архитектурный стиль](adr/ADR-03.md)
