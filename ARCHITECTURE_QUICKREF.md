# Краткий справочник: Архитектура Web-Search Multi-agent

## 🎯 Основная идея

GPT-Researcher использует **мульти-агентную архитектуру** с паттерном **Оркестратор**, где специализированные AI-агенты совместно проводят глубинное исследование любой темы.

## 📊 Ключевые паттерны

| Паттерн | Описание | Реализация |
|---------|----------|------------|
| **Orchestrator** | Координация команды агентов | `ChiefEditorAgent` |
| **Pipeline** | Последовательная обработка | Browser → Planner → Researcher → Writer → Publisher |
| **Strategy** | Взаимозаменяемые стратегии | Каждый агент = отдельная стратегия |
| **State Management** | Централизованное состояние | LangGraph `ResearchState` |
| **Parallel Processing** | Одновременное выполнение | asyncio.gather для секций |
| **Review-Revision Loop** | Контроль качества | Researcher → Reviewer → Reviser |
| **Human-in-the-Loop** | Участие человека | `HumanAgent` + условные переходы |
| **Facade** | Упрощённый интерфейс | `GPTResearcher` класс |

## 🤖 Агенты (8 штук)

### 1️⃣ ChiefEditorAgent (Оркестратор)
```python
# Расположение: /multi_agents/agents/orchestrator.py
# Роль: Мастер-координатор всей системы
chief = ChiefEditorAgent(task, websocket, stream_output)
graph = chief.init_research_team()
```

### 2️⃣ ResearchAgent (Исследователь)
```python
# Расположение: /multi_agents/agents/researcher.py
# Роль: Сбор информации из веб/документов
# Использует: GPTResearcher как инструмент
```

### 3️⃣ EditorAgent (Редактор/Планировщик)
```python
# Расположение: /multi_agents/agents/editor.py
# Роль: Планирование структуры + координация параллельных исследований
await editor.plan_research(state)  # Создаёт outline
await editor.run_parallel_research(state)  # Запускает исследование секций
```

### 4️⃣ WriterAgent (Писатель)
```python
# Расположение: /multi_agents/agents/writer.py
# Роль: Написание введения, заключения, оглавления
```

### 5️⃣ ReviewerAgent (Рецензент)
```python
# Расположение: /multi_agents/agents/reviewer.py
# Роль: Проверка соответствия guidelines
# Возвращает: None (одобрено) или feedback (доработать)
```

### 6️⃣ ReviserAgent (Корректор)
```python
# Расположение: /multi_agents/agents/reviser.py
# Роль: Исправление черновиков на основе feedback
```

### 7️⃣ PublisherAgent (Издатель)
```python
# Расположение: /multi_agents/agents/publisher.py
# Роль: Публикация в PDF/DOCX/Markdown
```

### 8️⃣ HumanAgent (Человек)
```python
# Расположение: /multi_agents/agents/human.py
# Роль: Интерфейс для человеческого участия
```

## 🔄 Workflow (6 фаз)

### Фаза 1: Browser (Первичное исследование)
```
ResearchAgent → GPTResearcher → Web Search → initial_research
```

### Фаза 2: Planning (Планирование)
```
EditorAgent → LLM → {title, date, sections[]}
```

### Фаза 3: Human Review (Опционально)
```
HumanAgent → 
  если feedback == None: продолжить
  если feedback exists: вернуться к Planning
```

### Фаза 4: Deep Research (Глубинное исследование)
```
EditorAgent координирует:
  ┌─ Section 1: Research → Review → (Revise)* → Approved ─┐
  ├─ Section 2: Research → Review → (Revise)* → Approved ─┤
  └─ Section N: Research → Review → (Revise)* → Approved ─┘
                    ↓
            research_data[]
```

### Фаза 5: Writing (Написание)
```
WriterAgent → LLM → {
  introduction,
  table_of_contents,
  conclusion,
  sources[]
}
```

### Фаза 6: Publishing (Публикация)
```
PublisherAgent → 
  ├─ Markdown (.md)
  ├─ PDF (.pdf)
  └─ Word (.docx)
```

## 📦 Состояния (TypedDict)

### ResearchState (главное)
```python
{
  "task": dict,              # Конфигурация
  "initial_research": str,   # Первичное исследование
  "sections": List[str],     # План секций
  "research_data": List,     # Данные по секциям
  "human_feedback": str,     # Feedback человека
  "title": str,              # Заголовок
  "date": str,               # Дата
  "introduction": str,       # Введение
  "conclusion": str,         # Заключение
  "table_of_contents": str,  # Оглавление
  "sources": List[str],      # Источники
  "report": str              # Финальный отчёт
}
```

### DraftState (для подзадач)
```python
{
  "task": dict,           # Конфигурация
  "topic": str,           # Тема секции
  "draft": dict,          # Черновик
  "review": str,          # Feedback рецензента
  "revision_notes": str   # Заметки о ревизии
}
```

## 🔧 Ключевые компоненты

### LangGraph (от LangChain)
```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(ResearchState)
workflow.add_node("browser", func)
workflow.add_edge('browser', 'planner')
workflow.add_conditional_edges('reviewer', condition, mapping)
chain = workflow.compile()
result = await chain.ainvoke({"task": task})
```

### GPTResearcher Core
```python
from gpt_researcher import GPTResearcher

researcher = GPTResearcher(
    query="Is AI in a hype cycle?",
    report_type="research_report",
    verbose=True
)
await researcher.conduct_research()
report = await researcher.write_report()
```

### Retrievers (Поисковики)
- **Tavily** — AI-специализированный поиск
- **Bing** — Microsoft Search API
- **Google** — Google Search API  
- **DuckDuckGo** — Privacy-friendly
- **Serper** — Google альтернатива
- **Searx** — Метапоисковик
- **MCP** — Model Context Protocol

## ⚙️ Конфигурация (task.json)

```json
{
  "query": "Is AI in a hype cycle?",
  "model": "gpt-4o",
  "max_sections": 3,
  "publish_formats": {
    "markdown": true,
    "pdf": true,
    "docx": true
  },
  "include_human_feedback": false,
  "source": "web",
  "follow_guidelines": true,
  "guidelines": [
    "The report MUST be written in APA format",
    "Each section MUST include supporting sources"
  ],
  "verbose": true
}
```

## 🚀 Запуск

```bash
# 1. Установка зависимостей
pip install -r multi_agents/requirements.txt

# 2. Настройка переменных окружения
export OPENAI_API_KEY="your-key"
export TAVILY_API_KEY="your-key"

# 3. Редактирование task.json
nano multi_agents/task.json

# 4. Запуск
cd multi_agents
python main.py
```

## 📂 Структура проекта

```
gpt-researcher-research/
├── multi_agents/              # Мульти-агентный слой
│   ├── agents/
│   │   ├── orchestrator.py   # ChiefEditorAgent
│   │   ├── researcher.py     # ResearchAgent
│   │   ├── editor.py         # EditorAgent
│   │   ├── writer.py         # WriterAgent
│   │   ├── reviewer.py       # ReviewerAgent
│   │   ├── reviser.py        # ReviserAgent
│   │   ├── publisher.py      # PublisherAgent
│   │   └── human.py          # HumanAgent
│   ├── memory/
│   │   ├── research.py       # ResearchState
│   │   └── draft.py          # DraftState
│   ├── task.json             # Конфигурация задачи
│   └── main.py               # Точка входа
│
└── gpt_researcher/            # Базовый слой
    ├── agent.py               # GPTResearcher класс
    ├── skills/                # Исследовательские навыки
    ├── retrievers/            # Веб-поисковики
    ├── llm_provider/          # LLM провайдеры
    └── vector_store/          # Векторные хранилища
```

## 💡 Преимущества архитектуры

| Преимущество | Объяснение |
|--------------|------------|
| **Модульность** | Каждый агент — независимый модуль |
| **Масштабируемость** | Легко добавить новых агентов |
| **Параллелизм** | Секции исследуются одновременно |
| **Качество** | Review-revision loops |
| **Гибкость** | Поддержка разных источников/форматов |
| **Контроль** | Human-in-the-loop опция |

## 🔑 Ключевые инсайты

1. **Двухуровневая архитектура:** 
   - Уровень 1: GPTResearcher (single agent)
   - Уровень 2: Multi-agent orchestration

2. **Параллельная обработка критична:**
   - Все секции исследуются одновременно
   - Использование `asyncio.gather()`

3. **Качество через циклы:**
   - Reviewer проверяет каждую секцию
   - Reviser исправляет до одобрения

4. **LangGraph управляет всем:**
   - StateGraph определяет workflow
   - Автоматическое управление состоянием
   - Условные переходы для гибкости

5. **GPTResearcher — это инструмент:**
   - ResearchAgent использует GPTResearcher
   - GPTResearcher работает автономно
   - Мульти-агентный слой координирует GPTResearcher

## 📚 Дополнительные ресурсы

- **Полная документация:** `ARCHITECTURE_RU.md`
- **Диаграммы:** `ARCHITECTURE_DIAGRAM.md`
- **Код:** `/multi_agents/` и `/gpt_researcher/`
- **README:** `/multi_agents/README.md`

## 🔗 Связь с научными работами

Архитектура вдохновлена:
- **STORM** (Stanford) — multi-agent research
- **Plan-and-Solve** — планирование и выполнение
- **RAG** (Retrieval-Augmented Generation) — использование внешних источников

---

**Вывод:** GPT-Researcher реализует **оркестраторную мульти-агентную архитектуру** с **параллельной обработкой**, **циклами контроля качества** и **управлением состоянием через LangGraph** для создания глубоких исследовательских отчётов.
