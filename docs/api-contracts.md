# Спецификация REST API

Базовый URL: `http://localhost:8000`

## Эндпоинты

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST | `/chat` | RAG-запрос |
| POST | `/agent` | Agentic RAG-запрос |
| GET | `/health` | Проверка работоспособности |
| GET | `/metrics` | Метрики Prometheus (OpenMetrics) |

---

## POST /chat

### Схемы

```python
from pydantic import BaseModel, Field


class Source(BaseModel):
    title: str = Field(..., description="Заголовок источника")
    url: str = Field(..., description="URL источника")
    score: float = Field(..., ge=0.0, le=1.0, description="Релевантность")


class ChatRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=2000)
    top_k: int = Field(4, ge=1, le=10)


class ChatResponse(BaseModel):
    answer: str = Field(..., description="Сгенерированный ответ")
    sources: list[Source] = Field(default_factory=list)
    latency_ms: float = Field(..., ge=0.0)
```

### Пример запроса

```json
{
  "question": "Как настроить max_depth в DecisionTreeClassifier?",
  "top_k": 4
}
```

### Пример ответа

```json
{
  "answer": "Параметр max_depth задаёт максимальную глубину дерева...",
  "sources": [
    {
      "title": "sklearn.tree.DecisionTreeClassifier",
      "url": "https://scikit-learn.org/stable/modules/generated/sklearn.tree.DecisionTreeClassifier.html",
      "score": 0.91
    }
  ],
  "latency_ms": 3420.5
}
```

---

## POST /agent

### Схемы

```python
from typing import Literal
from pydantic import BaseModel, Field


class TraceStep(BaseModel):
    iteration: int = Field(..., ge=0)
    tool: Literal["documentation_search", "python_repl", "web_search"] | None = None
    input: str = Field(..., description="Вход инструмента")
    output: str = Field(..., description="Выход инструмента")
    latency_ms: float = Field(..., ge=0.0)


class AgentRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=2000)
    thread_id: str = Field(..., description="ID сессии для checkpointing")


class AgentResponse(BaseModel):
    answer: str
    trace: list[TraceStep] = Field(default_factory=list)
    sources: list[Source] = Field(default_factory=list)
    iterations: int = Field(..., ge=0)
    latency_ms: float = Field(..., ge=0.0)
```

### Пример запроса

```json
{
  "question": "Посчитай accuracy для [1,0,1,1] и [1,0,0,1]",
  "thread_id": "sess-42"
}
```

### Пример ответа

```json
{
  "answer": "Accuracy = 0.75",
  "trace": [
    {
      "iteration": 1,
      "tool": "python_repl",
      "input": "from sklearn.metrics import accuracy_score; accuracy_score([1,0,1,1],[1,0,0,1])",
      "output": "0.75",
      "latency_ms": 210.4
    }
  ],
  "sources": [],
  "iterations": 2,
  "latency_ms": 5120.0
}
```

---

## GET /health

### Пример ответа

```json
{ "status": "ok" }
```

---

## GET /metrics

Возвращает метрики в формате **OpenMetrics** (Prometheus).

### Пример ответа

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="POST",endpoint="/chat",status="200"} 42

# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{endpoint="/chat",le="1.0"} 10
http_request_duration_seconds_bucket{endpoint="/chat",le="5.0"} 38
http_request_duration_seconds_bucket{endpoint="/chat",le="+Inf"} 42
```

---

## Ошибки

### 422 Unprocessable Entity

Возвращается при невалидном теле запроса (Pydantic-валидация).

```json
{
  "detail": [
    {
      "type": "string_too_short",
      "loc": ["body", "question"],
      "msg": "String should have at least 1 character",
      "input": ""
    }
  ]
}
```

### 500 Internal Server Error

Возвращается при непредвиденных ошибках. Тело содержит `detail` с описанием.
