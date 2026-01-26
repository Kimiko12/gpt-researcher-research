# Архитектура Web-Search Multi-agent системы GPT-Researcher

## Оглавление
1. [Введение](#введение)
2. [Общая архитектура](#общая-архитектура)
3. [Паттерны проектирования](#паттерны-проектирования)
4. [Агенты и их роли](#агенты-и-их-роли)
5. [Рабочий процесс](#рабочий-процесс)
6. [Управление состоянием](#управление-состоянием)
7. [Ключевые компоненты](#ключевые-компоненты)

---

## Введение

GPT-Researcher — это автономный агент для глубинного исследования, который работает как в веб-среде, так и с локальными документами. Система использует **мульти-агентную архитектуру** на основе LangGraph для координации специализированных AI-агентов, которые совместно выполняют комплексные исследовательские задачи.

---

## Общая архитектура

### Двухуровневая архитектура

Проект реализует **двухуровневую мульти-агентную архитектуру**:

#### Уровень 1: Базовый GPTResearcher (Single Agent)
Находится в директории `/gpt_researcher/` и реализует одного самодостаточного агента-исследователя, который:
- Выполняет веб-поиск через различные ретриверы (Tavily, DuckDuckGo, Bing, Google, Serper, Searx, MCP)
- Собирает и обрабатывает информацию из источников
- Генерирует исследовательские отчеты
- Управляет контекстом и памятью

#### Уровень 2: Multi-Agent Orchestration
Находится в директории `/multi_agents/` и реализует **оркестрационный слой** с командой специализированных агентов, использующих базовый GPTResearcher как инструмент.

---

## Паттерны проектирования

### 1. **Orchestrator Pattern (Паттерн Оркестратор)**

Центральный паттерн архитектуры — это **Orchestrator** (Дирижёр/Координатор).

**Реализация:** Класс `ChiefEditorAgent` в `/multi_agents/agents/orchestrator.py`

```python
class ChiefEditorAgent:
    """Агент, ответственный за управление и координацию задач редактирования"""
    
    def init_research_team(self):
        """Инициализация и создание рабочего процесса для исследовательской команды"""
        agents = self._initialize_agents()
        return self._create_workflow(agents)
```

**Ключевые характеристики:**
- Координирует работу всех остальных агентов
- Управляет жизненным циклом исследования
- Создает и контролирует граф состояний (StateGraph)
- Определяет последовательность выполнения задач

### 2. **Pipeline Pattern (Паттерн Конвейер)**

Исследование проходит через **последовательный конвейер** этапов:

```
Browser → Planner → Human (опционально) → Researcher → Writer → Publisher
```

**Реализация через LangGraph StateGraph:**

```python
def _create_workflow(self, agents):
    workflow = StateGraph(ResearchState)
    
    # Добавление узлов для каждого агента
    workflow.add_node("browser", agents["research"].run_initial_research)
    workflow.add_node("planner", agents["editor"].plan_research)
    workflow.add_node("researcher", agents["editor"].run_parallel_research)
    workflow.add_node("writer", agents["writer"].run)
    workflow.add_node("publisher", agents["publisher"].run)
    workflow.add_node("human", agents["human"].review_plan)
    
    # Определение рёбер (переходов)
    workflow.add_edge('browser', 'planner')
    workflow.add_edge('planner', 'human')
    workflow.add_edge('researcher', 'writer')
    workflow.add_edge('writer', 'publisher')
```

### 3. **Strategy Pattern (Паттерн Стратегия)**

Каждый агент — это **отдельная стратегия** для выполнения специфической задачи:

- **ResearchAgent** — стратегия сбора информации
- **EditorAgent** — стратегия планирования и структурирования
- **WriterAgent** — стратегия написания отчёта
- **ReviewerAgent** — стратегия валидации качества
- **ReviserAgent** — стратегия исправления недостатков
- **PublisherAgent** — стратегия публикации в различных форматах

### 4. **State Management Pattern (Паттерн Управления Состоянием)**

Использует **централизованное хранилище состояния** на основе TypedDict:

```python
class ResearchState(TypedDict):
    task: dict                    # Параметры задачи
    initial_research: str         # Первичное исследование
    sections: List[str]           # Секции отчёта
    research_data: List[dict]     # Собранные данные
    human_feedback: str           # Обратная связь от человека
    title: str                    # Заголовок
    headers: dict                 # Заголовки секций
    date: str                     # Дата
    table_of_contents: str        # Оглавление
    introduction: str             # Введение
    conclusion: str               # Заключение
    sources: List[str]            # Источники
    report: str                   # Финальный отчёт
```

Каждый агент получает состояние, обрабатывает его и возвращает обновлённое состояние.

### 5. **Parallel Processing Pattern (Паттерн Параллельной Обработки)**

**Критически важный паттерн** для производительности:

```python
async def run_parallel_research(self, research_state: Dict[str, any]):
    """Выполнение параллельных исследовательских задач для каждой секции"""
    queries = research_state.get("sections")
    
    # Создание параллельных задач
    final_drafts = [
        chain.ainvoke(self._create_task_input(research_state, query, title))
        for query in queries
    ]
    
    # Ожидание завершения всех задач параллельно
    research_results = [
        result["draft"] for result in await asyncio.gather(*final_drafts)
    ]
```

### 6. **Review-Revision Loop Pattern (Паттерн Цикла Проверки-Исправления)**

Для каждой подтемы реализован **цикл качества**:

```python
workflow.add_conditional_edges(
    "reviewer",
    lambda draft: "accept" if draft["review"] is None else "revise",
    {"accept": END, "revise": "reviser"},
)
```

Процесс:
1. **Researcher** создаёт черновик
2. **Reviewer** проверяет соответствие требованиям
3. Если не соответствует → **Reviser** исправляет → возврат к Reviewer
4. Если соответствует → переход к следующему этапу

### 7. **Human-in-the-Loop Pattern (Паттерн Человека в Цикле)**

Опциональная возможность **человеческого участия**:

```python
workflow.add_conditional_edges(
    'human',
    lambda review: "accept" if review['human_feedback'] is None else "revise",
    {"accept": "researcher", "revise": "planner"}
)
```

Человек может:
- Одобрить план исследования
- Запросить изменения в структуре отчёта

### 8. **Facade Pattern (Паттерн Фасад)**

**GPTResearcher** служит **фасадом** для сложной системы:

```python
researcher = GPTResearcher(
    query=query, 
    report_type="research_report",
    verbose=True
)
await researcher.conduct_research()
report = await researcher.write_report()
```

Скрывает внутреннюю сложность:
- Управление ретриверами
- Работу с векторными хранилищами
- Обработку контекста
- Взаимодействие с LLM

---

## Агенты и их роли

### 1. **ChiefEditorAgent** (Главный Редактор) - ОРКЕСТРАТОР

**Расположение:** `/multi_agents/agents/orchestrator.py`

**Роль:** Мастер-агент, координирующий всю команду

**Ответственность:**
- Инициализация всех агентов
- Создание графа рабочего процесса (workflow)
- Управление жизненным циклом исследования
- Создание выходных директорий
- Конфигурация системы

**Ключевые методы:**
- `init_research_team()` — создание команды
- `run_research_task()` — запуск исследования
- `_create_workflow()` — построение графа задач

### 2. **ResearchAgent** (Агент-Исследователь)

**Расположение:** `/multi_agents/agents/researcher.py`

**Роль:** Сбор информации из веб-источников и документов

**Использует:** Класс `GPTResearcher` как основной инструмент

**Ответственность:**
- Начальное исследование по главному запросу (`run_initial_research`)
- Глубинное исследование по подтемам (`run_depth_research`)
- Работа с различными источниками (web/local)
- Генерация отчётов по подтемам

**Типы отчётов:**
- `research_report` — основной отчёт
- `subtopic_report` — отчёт по подтеме

### 3. **EditorAgent** (Агент-Редактор/Планировщик)

**Расположение:** `/multi_agents/agents/editor.py`

**Роль:** Планирование структуры исследования

**Ответственность:**
- Анализ начального исследования
- Создание плана отчёта (outline)
- Генерация секций и подтем
- Координация параллельного исследования
- Управление подграфом исследования каждой секции

**Ключевые методы:**
- `plan_research()` — создание плана
- `run_parallel_research()` — запуск параллельных исследований

**Создаёт подграф для каждой секции:**
```
Researcher → Reviewer → (Reviser → Reviewer)* → END
```

### 4. **WriterAgent** (Агент-Писатель)

**Расположение:** `/multi_agents/agents/writer.py`

**Роль:** Написание финального отчёта

**Ответственность:**
- Генерация введения
- Создание оглавления
- Написание заключения
- Компиляция списка источников
- Адаптация под требования (guidelines)

**Выходные данные:**
- `introduction` — введение с гиперссылками
- `table_of_contents` — оглавление
- `conclusion` — заключение
- `sources` — форматированный список источников

### 5. **ReviewerAgent** (Агент-Рецензент)

**Расположение:** `/multi_agents/agents/reviewer.py`

**Роль:** Контроль качества исследования

**Ответственность:**
- Проверка соответствия guidelines
- Валидация полноты информации
- Генерация feedback для ревизии
- Принятие/отклонение черновиков

**Логика работы:**
```python
if draft meets guidelines:
    return None  # принять
else:
    return "feedback for revision"  # отправить на доработку
```

### 6. **ReviserAgent** (Агент-Корректор)

**Расположение:** `/multi_agents/agents/reviser.py`

**Роль:** Исправление черновиков на основе feedback

**Ответственность:**
- Анализ замечаний рецензента
- Переписывание черновика
- Сохранение стиля и структуры
- Документирование изменений

**Выходные данные:**
```json
{
  "draft": "исправленный черновик",
  "revision_notes": "описание внесённых изменений"
}
```

### 7. **PublisherAgent** (Агент-Издатель)

**Расположение:** `/multi_agents/agents/publisher.py`

**Роль:** Публикация отчёта в различных форматах

**Ответственность:**
- Сборка финального отчёта из всех частей
- Форматирование в Markdown
- Экспорт в PDF
- Экспорт в DOCX
- Сохранение файлов

**Генерирует структуру:**
```markdown
# Заголовок
## Введение
## Оглавление
## Секции (из исследований)
## Заключение
## Ссылки
```

### 8. **HumanAgent** (Агент-Человек)

**Расположение:** `/multi_agents/agents/human.py`

**Роль:** Интерфейс для участия человека

**Ответственность:**
- Получение плана от Editor
- Запрос обратной связи от пользователя
- Передача feedback обратно в систему

**Режимы:**
- `include_human_feedback: true` — ждёт ввода пользователя
- `include_human_feedback: false` — автоматическое одобрение

---

## Рабочий процесс

### Этап 1: Инициализация (Browser Phase)

**Агент:** ResearchAgent

**Действия:**
1. Получение основного query из задачи
2. Запуск `GPTResearcher` для первичного исследования
3. Сбор общей информации по теме
4. Создание summary начального исследования

**Выход:** `initial_research` — краткое изложение найденной информации

### Этап 2: Планирование (Planning Phase)

**Агент:** EditorAgent

**Действия:**
1. Анализ `initial_research`
2. Определение ключевых подтем
3. Генерация структуры отчёта
4. Создание списка секций (max = `max_sections`)

**LLM Prompt:**
```
"Ваша задача — сгенерировать outline заголовков секций 
на основе summary исследования. Максимум {max_sections} секций.
Фокус ТОЛЬКО на исследовательских темах, НЕ включайте 
введение, заключение и ссылки."
```

**Выход:** 
```json
{
  "title": "Заголовок исследования",
  "date": "текущая дата",
  "sections": ["Секция 1", "Секция 2", "Секция 3"]
}
```

### Этап 3: Human Review (опционально)

**Агент:** HumanAgent

**Действия:**
1. Предоставление плана пользователю
2. Ожидание обратной связи
3. Возврат к планированию ИЛИ продолжение

**Условная логика:**
- `human_feedback is None` → продолжить исследование
- `human_feedback exists` → вернуться к планированию с feedback

### Этап 4: Глубинное Исследование (Deep Research Phase)

**Агент:** EditorAgent (координатор) + ResearchAgent (исполнитель)

**Действия для КАЖДОЙ секции (параллельно):**

#### 4.1. Исследование подтемы
- ResearchAgent создаёт подробный отчёт по секции
- Использует parent_query для контекста
- Генерирует `subtopic_report`

#### 4.2. Проверка качества (Review)
- ReviewerAgent проверяет черновик
- Сравнивает с guidelines
- Возвращает feedback или None

#### 4.3. Ревизия (если нужна)
- ReviserAgent исправляет черновик
- Учитывает замечания рецензента
- Возврат к ReviewerAgent

#### 4.4. Завершение секции
- Цикл продолжается до одобрения
- Финальный черновик сохраняется

**Параллельное выполнение:**
```python
# Все секции исследуются одновременно
tasks = [research_section(s) for s in sections]
results = await asyncio.gather(*tasks)
```

**Выход:** `research_data` — список отчётов по всем секциям

### Этап 5: Написание Отчёта (Writing Phase)

**Агент:** WriterAgent

**Действия:**
1. Получение всех `research_data`
2. Анализ содержимого всех секций
3. Генерация введения (с контекстом всего исследования)
4. Создание оглавления (на основе структуры)
5. Написание заключения (обобщение всех данных)
6. Компиляция списка источников (дедупликация)
7. Адаптация под guidelines (если указано)

**LLM генерирует:**
```json
{
  "table_of_contents": "- Секция 1\n- Секция 2\n...",
  "introduction": "Введение с [гиперссылками](url)",
  "conclusion": "Заключение с [ссылками](url)",
  "sources": ["- Источник 1 [url](url)", "..."]
}
```

**Выход:** Структурированные компоненты финального отчёта

### Этап 6: Публикация (Publishing Phase)

**Агент:** PublisherAgent

**Действия:**
1. Сборка всех компонентов в единый документ
2. Форматирование в Markdown
3. Экспорт в запрошенные форматы:
   - **PDF** — через markdown → PDF конвертер
   - **DOCX** — через markdown → Word конвертер
   - **Markdown** — сохранение .md файла

**Структура финального документа:**
```markdown
# {title}
#### Date: {date}

## Introduction
{introduction}

## Table of Contents
{table_of_contents}

{section_1_content}

{section_2_content}

...

## Conclusion
{conclusion}

## References
{sources}
```

**Выход:** Файлы отчёта в `/outputs/run_{task_id}_{query}/`

---

## Управление состоянием

### ResearchState (Главное состояние)

Используется на верхнем уровне workflow:

```python
class ResearchState(TypedDict):
    task: dict                    # Конфигурация задачи
    initial_research: str         # Результат browser phase
    sections: List[str]           # План от editor
    research_data: List[dict]     # Результаты researcher
    human_feedback: str           # Feedback от human
    title: str                    # Заголовок отчёта
    headers: dict                 # Заголовки секций
    date: str                     # Дата генерации
    table_of_contents: str        # Оглавление
    introduction: str             # Введение
    conclusion: str               # Заключение
    sources: List[str]            # Список источников
    report: str                   # Финальный отчёт
```

### DraftState (Состояние для подзадач)

Используется в подграфе исследования каждой секции:

```python
class DraftState(TypedDict):
    task: dict                    # Конфигурация задачи
    topic: str                    # Тема секции
    draft: dict                   # Черновик отчёта
    review: str                   # Feedback от reviewer
    revision_notes: str           # Заметки о ревизии
```

### Принципы работы с состоянием

1. **Иммутабельность:** Каждый агент возвращает новое состояние, а не изменяет существующее
2. **Частичные обновления:** Агенты возвращают только изменённые поля
3. **Автоматическое слияние:** LangGraph автоматически объединяет изменения
4. **Type Safety:** TypedDict обеспечивает типобезопасность

### Передача состояния между агентами

```python
# Browser создаёт initial_research
{"initial_research": "текст исследования"}

# Planner добавляет title, date, sections
{"title": "...", "date": "...", "sections": [...]}

# Researcher добавляет research_data
{"research_data": [{...}, {...}]}

# Writer добавляет introduction, conclusion, etc.
{"introduction": "...", "conclusion": "...", ...}

# Publisher добавляет report
{"report": "финальный markdown"}
```

---

## Ключевые компоненты

### LangGraph Integration

**Библиотека:** LangGraph от LangChain

**Назначение:** Построение stateful multi-actor applications

**Основные концепции:**

1. **StateGraph:**
```python
workflow = StateGraph(ResearchState)
```
Граф состояний, где узлы — функции, рёбра — переходы

2. **Узлы (Nodes):**
```python
workflow.add_node("browser", agent.run_initial_research)
```
Каждый узел — async функция, принимающая и возвращающая состояние

3. **Рёбра (Edges):**
```python
workflow.add_edge('browser', 'planner')  # Безусловный переход
```

4. **Условные рёбра (Conditional Edges):**
```python
workflow.add_conditional_edges(
    'reviewer',
    lambda draft: "accept" if draft["review"] is None else "revise",
    {"accept": END, "revise": "reviser"}
)
```

5. **Компиляция и выполнение:**
```python
chain = workflow.compile()
result = await chain.ainvoke({"task": task})
```

### GPTResearcher Core

**Расположение:** `/gpt_researcher/agent.py`

**Класс:** `GPTResearcher`

**Основные компоненты:**

1. **ResearchConductor:**
   - Управление процессом исследования
   - Выбор агента на основе типа отчёта
   - Обработка query

2. **BrowserManager:**
   - Веб-скрапинг
   - Обработка JavaScript
   - Фильтрация контента

3. **SourceCurator:**
   - Оценка релевантности источников
   - Дедупликация
   - Фильтрация качества

4. **ContextManager:**
   - Управление контекстом для LLM
   - Сжатие информации
   - Приоритизация данных

5. **ReportGenerator:**
   - Генерация финальных отчётов
   - Форматирование
   - Добавление ссылок

6. **Memory:**
   - Векторное хранилище для embeddings
   - Семантический поиск по контексту
   - Кэширование

### Retrievers (Поисковики)

**Расположение:** `/gpt_researcher/retrievers/`

**Поддерживаемые сервисы:**
- **Tavily** — специализированный AI search
- **Bing** — Microsoft Bing Search API
- **Google** — Google Search API
- **DuckDuckGo** — privacy-friendly поиск
- **Serper** — Google Search API альтернатива
- **Searx** — метапоисковик
- **MCP (Model Context Protocol)** — расширяемый протокол для инструментов

**Архитектура:**
```python
retrievers = get_retrievers(headers, cfg)
search_results = await get_search_results(
    query=query,
    retrievers=retrievers,
    max_results=10
)
```

### LLM Provider System

**Расположение:** `/gpt_researcher/llm_provider/`

**Назначение:** Абстракция для работы с различными LLM

**Поддерживаемые провайдеры:**
- OpenAI (GPT-4, GPT-3.5)
- Anthropic (Claude)
- Google (Gemini)
- Azure OpenAI
- Ollama (локальные модели)
- И другие через LangChain

**Использование:**
```python
llm_provider = GenericLLMProvider()
response = await llm_provider.call_model(
    prompt=prompt,
    model="gpt-4o",
    response_format="json"
)
```

### Vector Store Integration

**Расположение:** `/gpt_researcher/vector_store/`

**Назначение:** Хранение и поиск embeddings

**Поддержка:**
- Chroma
- Pinecone
- Weaviate
- LanceDB
- Custom векторные БД

**Использование:**
```python
vector_store = VectorStoreWrapper(vector_store_instance)
relevant_docs = await vector_store.search(query, k=5)
```

---

## Заключение

### Преимущества архитектуры

1. **Модульность:** Каждый агент — независимый модуль со своей ответственностью
2. **Масштабируемость:** Легко добавить новых агентов или изменить workflow
3. **Параллелизм:** Секции исследуются одновременно, ускоряя процесс
4. **Качество:** Цикл review-revision обеспечивает соответствие требованиям
5. **Гибкость:** Поддержка различных источников, форматов, LLM-провайдеров
6. **Human-in-the-Loop:** Опциональное участие человека для контроля

### Ключевые паттерны

- **Orchestrator** — ChiefEditorAgent координирует всю систему
- **Pipeline** — последовательная обработка через этапы
- **Strategy** — каждый агент реализует свою стратегию
- **State Management** — централизованное управление состоянием через LangGraph
- **Parallel Processing** — одновременное исследование секций
- **Review-Revision Loop** — контроль качества через цикл обратной связи
- **Facade** — GPTResearcher скрывает сложность базового исследования

### Технологический стек

- **LangGraph** — построение stateful multi-agent workflows
- **LangChain** — интеграция с LLM и инструментами
- **AsyncIO** — асинхронное выполнение задач
- **TypedDict** — типобезопасность состояний
- **GPTResearcher** — базовый исследовательский движок

Эта архитектура позволяет создавать глубокие, качественные исследовательские отчёты, комбинируя преимущества специализации агентов, параллельной обработки и контроля качества через review процессы.
