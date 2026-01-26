# Диаграммы архитектуры GPT-Researcher Multi-Agent System

## 1. Общая архитектура системы

```mermaid
graph TB
    subgraph "Multi-Agent Orchestration Layer"
        Chief[ChiefEditorAgent<br/>Оркестратор]
        Browser[Browser Phase<br/>ResearchAgent]
        Planner[Planning Phase<br/>EditorAgent]
        Human[Human Review<br/>HumanAgent]
        Researcher[Research Phase<br/>EditorAgent + ResearchAgent]
        Writer[Writing Phase<br/>WriterAgent]
        Publisher[Publishing Phase<br/>PublisherAgent]
        
        Chief -->|инициализация| Browser
        Browser -->|initial_research| Planner
        Planner -->|план отчёта| Human
        Human -->|одобрение| Researcher
        Human -->|feedback| Planner
        Researcher -->|research_data| Writer
        Writer -->|структурированный контент| Publisher
        Publisher -->|финальный отчёт| Output[📄 PDF/DOCX/MD]
    end
    
    subgraph "Core GPTResearcher Layer"
        GPT[GPTResearcher]
        Retrievers[Web Retrievers<br/>Tavily/Bing/Google/etc]
        Scraper[BrowserManager]
        Context[ContextManager]
        Memory[Memory/VectorStore]
        LLM[LLM Provider]
        
        GPT --> Retrievers
        GPT --> Scraper
        GPT --> Context
        GPT --> Memory
        GPT --> LLM
    end
    
    Browser -.использует.-> GPT
    Researcher -.использует.-> GPT
    
    style Chief fill:#ff6b6b
    style GPT fill:#4ecdc4
    style Output fill:#95e1d3
```

## 2. Workflow с управлением состоянием (LangGraph StateGraph)

```mermaid
stateDiagram-v2
    [*] --> Browser
    
    Browser --> Planner: initial_research
    
    Planner --> Human: title, sections, date
    
    Human --> Researcher: human_feedback = None
    Human --> Planner: human_feedback exists
    
    state Researcher {
        [*] --> ParallelResearch
        
        state ParallelResearch {
            [*] --> Section1
            [*] --> Section2
            [*] --> Section3
            
            state Section1 {
                [*] --> Research1
                Research1 --> Review1
                Review1 --> Revise1: не соответствует
                Revise1 --> Review1
                Review1 --> [*]: соответствует
            }
            
            state Section2 {
                [*] --> Research2
                Research2 --> Review2
                Review2 --> Revise2: не соответствует
                Revise2 --> Review2
                Review2 --> [*]: соответствует
            }
            
            state Section3 {
                [*] --> Research3
                Research3 --> Review3
                Review3 --> Revise3: не соответствует
                Revise3 --> Review3
                Review3 --> [*]: соответствует
            }
        }
        
        ParallelResearch --> [*]
    }
    
    Researcher --> Writer: research_data[]
    Writer --> Publisher: introduction, conclusion, sources
    Publisher --> [*]: report
```

## 3. Архитектура агентов и их взаимодействие

```mermaid
graph LR
    subgraph "Orchestration"
        Chief[ChiefEditorAgent<br/>🎭 Координатор]
    end
    
    subgraph "Research Team"
        Research[ResearchAgent<br/>🔍 Исследователь]
        Editor[EditorAgent<br/>📋 Редактор/Планировщик]
        Writer[WriterAgent<br/>✍️ Писатель]
        Publisher[PublisherAgent<br/>📰 Издатель]
    end
    
    subgraph "Quality Control"
        Reviewer[ReviewerAgent<br/>👀 Рецензент]
        Reviser[ReviserAgent<br/>🔧 Корректор]
    end
    
    subgraph "Human Interface"
        Human[HumanAgent<br/>👤 Человек]
    end
    
    Chief -->|управляет| Research
    Chief -->|управляет| Editor
    Chief -->|управляет| Writer
    Chief -->|управляет| Publisher
    Chief -->|управляет| Human
    
    Editor -->|координирует| Research
    Editor -->|координирует| Reviewer
    Editor -->|координирует| Reviser
    
    Research -->|черновик| Reviewer
    Reviewer -->|feedback| Reviser
    Reviser -->|исправленный черновик| Reviewer
    Reviewer -->|одобрение| Editor
    
    style Chief fill:#ff6b6b
    style Research fill:#4ecdc4
    style Editor fill:#ffe66d
    style Writer fill:#a8e6cf
    style Publisher fill:#ffd3b6
    style Reviewer fill:#ffaaa5
    style Reviser fill:#ff8b94
    style Human fill:#c7ceea
```

## 4. Поток данных через состояния

```mermaid
graph TD
    Start([Начало: task]) --> S1[ResearchState]
    
    S1 -->|Browser| S2[+ initial_research]
    S2 -->|Planner| S3[+ title<br/>+ date<br/>+ sections]
    S3 -->|Human| S4{feedback?}
    S4 -->|нет| S5[без изменений]
    S4 -->|да| S3
    
    S5 -->|Researcher| Parallel[Параллельное исследование]
    
    Parallel --> D1[DraftState 1]
    Parallel --> D2[DraftState 2]
    Parallel --> D3[DraftState N]
    
    D1 --> Loop1{Review<br/>passed?}
    Loop1 -->|нет| Revise1[Revise]
    Revise1 --> Loop1
    Loop1 -->|да| Result1[draft 1]
    
    D2 --> Loop2{Review<br/>passed?}
    Loop2 -->|нет| Revise2[Revise]
    Revise2 --> Loop2
    Loop2 -->|да| Result2[draft 2]
    
    D3 --> Loop3{Review<br/>passed?}
    Loop3 -->|нет| Revise3[Revise]
    Revise3 --> Loop3
    Loop3 -->|да| Result3[draft N]
    
    Result1 --> S6[+ research_data]
    Result2 --> S6
    Result3 --> S6
    
    S6 -->|Writer| S7[+ introduction<br/>+ conclusion<br/>+ table_of_contents<br/>+ sources<br/>+ headers]
    
    S7 -->|Publisher| S8[+ report]
    
    S8 --> End([Конец: финальный отчёт])
    
    style Start fill:#95e1d3
    style End fill:#95e1d3
    style S1 fill:#f0f0f0
    style S2 fill:#e0e0e0
    style S3 fill:#d0d0d0
    style S6 fill:#c0c0c0
    style S7 fill:#b0b0b0
    style S8 fill:#a0a0a0
```

## 5. Паттерны проектирования

```mermaid
graph TB
    subgraph "Orchestrator Pattern"
        O[ChiefEditorAgent]
        O --> A1[Agent 1]
        O --> A2[Agent 2]
        O --> A3[Agent 3]
    end
    
    subgraph "Pipeline Pattern"
        P1[Stage 1] --> P2[Stage 2]
        P2 --> P3[Stage 3]
        P3 --> P4[Stage 4]
    end
    
    subgraph "Strategy Pattern"
        Context[Context]
        Context --> S1[Strategy 1<br/>ResearchAgent]
        Context --> S2[Strategy 2<br/>WriterAgent]
        Context --> S3[Strategy N<br/>PublisherAgent]
    end
    
    subgraph "Review-Revision Loop"
        Draft[Draft] --> Review{Review}
        Review -->|fail| Revise[Revise]
        Revise --> Review
        Review -->|pass| Accept[Accept]
    end
    
    style O fill:#ff6b6b
    style Context fill:#4ecdc4
    style Review fill:#ffe66d
```

## 6. GPTResearcher Core Architecture

```mermaid
graph TB
    subgraph "GPTResearcher Facade"
        GPT[GPTResearcher<br/>Главный класс]
    end
    
    subgraph "Research Skills"
        Conductor[ResearchConductor<br/>Управление исследованием]
        ReportGen[ReportGenerator<br/>Генерация отчётов]
        CtxMgr[ContextManager<br/>Управление контекстом]
        Browser[BrowserManager<br/>Веб-скрапинг]
        Curator[SourceCurator<br/>Кураторство источников]
        Deep[DeepResearchSkill<br/>Глубинное исследование]
    end
    
    subgraph "Infrastructure"
        Config[Config<br/>Конфигурация]
        Memory[Memory<br/>Векторное хранилище]
        LLM[LLM Provider<br/>Провайдер моделей]
        Retrievers[Retrievers<br/>Поисковики]
    end
    
    GPT --> Conductor
    GPT --> ReportGen
    GPT --> CtxMgr
    GPT --> Browser
    GPT --> Curator
    GPT --> Deep
    
    GPT --> Config
    GPT --> Memory
    GPT --> LLM
    GPT --> Retrievers
    
    Retrievers --> Tavily[Tavily API]
    Retrievers --> Bing[Bing Search]
    Retrievers --> Google[Google Search]
    Retrievers --> Duck[DuckDuckGo]
    Retrievers --> MCP[MCP Protocol]
    
    style GPT fill:#4ecdc4
    style Conductor fill:#ffe66d
    style Memory fill:#a8e6cf
    style LLM fill:#ffd3b6
```

## 7. Детальный процесс исследования одной секции

```mermaid
sequenceDiagram
    participant E as EditorAgent
    participant R as ResearchAgent
    participant G as GPTResearcher
    participant Rev as ReviewerAgent
    participant Rvs as ReviserAgent
    
    E->>R: run_depth_research(topic)
    activate R
    
    R->>G: conduct_research(subtopic_report)
    activate G
    
    G->>G: choose_agent()
    G->>G: get_search_results()
    G->>G: scrape_sources()
    G->>G: summarize_content()
    G->>G: write_report()
    
    G-->>R: subtopic_report
    deactivate G
    
    R-->>Rev: draft
    deactivate R
    activate Rev
    
    Rev->>Rev: review_draft(guidelines)
    
    alt Не соответствует
        Rev-->>Rvs: feedback
        deactivate Rev
        activate Rvs
        
        Rvs->>Rvs: revise_draft()
        Rvs-->>Rev: revised_draft + notes
        deactivate Rvs
        activate Rev
        
        Rev->>Rev: review_draft()
        
        alt Всё ещё не соответствует
            Rev-->>Rvs: more feedback
            Note over Rev,Rvs: Цикл повторяется
        else Соответствует
            Rev-->>E: approved_draft
        end
    else Соответствует
        Rev-->>E: approved_draft
        deactivate Rev
    end
```

## 8. MCP (Model Context Protocol) Integration

```mermaid
graph TB
    subgraph "GPTResearcher"
        Agent[Agent]
        Config[MCP Config]
    end
    
    subgraph "MCP Layer"
        Client[MCP Client]
        Selector[Tool Selector]
    end
    
    subgraph "MCP Servers"
        Server1[Search MCP]
        Server2[FileSystem MCP]
        Server3[Custom MCP]
    end
    
    subgraph "External Resources"
        Web[Web APIs]
        FS[File System]
        DB[Databases]
    end
    
    Agent --> Config
    Config --> Client
    Client --> Selector
    
    Selector --> Server1
    Selector --> Server2
    Selector --> Server3
    
    Server1 --> Web
    Server2 --> FS
    Server3 --> DB
    
    style Agent fill:#4ecdc4
    style Client fill:#ffe66d
    style Selector fill:#a8e6cf
```

## Примечания к диаграммам

### Условные обозначения:
- **Синие блоки** — основные компоненты (ChiefEditorAgent, GPTResearcher)
- **Жёлтые блоки** — агенты планирования и координации
- **Зелёные блоки** — агенты генерации контента
- **Красные/оранжевые блоки** — агенты контроля качества
- **Фиолетовые блоки** — человеческий интерфейс
- **Сплошные стрелки** — прямые вызовы/переходы
- **Пунктирные стрелки** — использование/зависимость

### Ключевые особенности архитектуры:

1. **Двухуровневая структура:** Оркестрационный слой (multi-agents) над базовым слоем (gpt-researcher)
2. **Параллельная обработка:** Секции исследуются одновременно для ускорения
3. **Циклы обратной связи:** Review-Revision loops обеспечивают качество
4. **Управление состоянием:** LangGraph StateGraph координирует поток данных
5. **Модульность:** Каждый агент — независимый модуль с чёткой ответственностью
