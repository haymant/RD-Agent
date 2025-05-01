# RD-Agent Design

## Architecture

```mermaid
classDiagram
    %% Core Framework
    class Scenario {
        <<abstract>>
        +get_source_data_desc()
        +get_scenario_all_desc()
        +source_data
    }

    class Developer {
        <<abstract>>
        +develop(exp)
    }

    class Evaluator {
        <<abstract>>
        +evaluate()
    }

    class EvoAgent {
        <<abstract>>
        +multistep_evolve()
    }

    class KnowledgeBase {
        +load()
        +dump()
    }

    %% Experiment and Task Components
    class Task {
        +name: str
        +version: int
        +get_task_information()
    }

    class Experiment {
        +task: Task
        +hypothesis: Hypothesis
        +workspace: Workspace
    }

    class Workspace {
        <<abstract>>
        +prepare()
        +execute()
        +all_codes
    }

    class Hypothesis {
        +content: str
        +description: str
    }

    %% Knowledge Management
    class Graph {
        +find_node()
        +batch_embedding()
    }

    class VectorBase {
        +add()
        +search()
    }

    class UndirectedGraph {
        +add_node()
        +semantic_search()
        +query_by_content()
    }

    class PDVectorBase {
        +add()
        +search()
    }

    %% Workflow Components
    class RDLoop {
        +PROP_SETTING
        +_propose()
        +_exp_gen()
        +coding()
        +running()
        +feedback()
    }

    %% Evolution Framework
    class EvolvingStrategy {
        <<abstract>>
        +evolve()
    }

    class RAGStrategy {
        <<abstract>>
        +query()
        +generate_knowledge()
    }

    %% Relationships
    KnowledgeBase <|-- Graph
    KnowledgeBase <|-- VectorBase
    Graph <|-- UndirectedGraph
    VectorBase <|-- PDVectorBase
    
    Experiment --> Task
    Experiment --> Hypothesis
    Experiment --> Workspace
    
    EvoAgent --> Evaluator
    EvoAgent --> EvolvingStrategy
    
    RDLoop --> Developer
    RDLoop --> Experiment
    RDLoop ..> Hypothesis
    
    Developer --> Scenario
    Developer --> Experiment
    
    RAGStrategy --> KnowledgeBase
```

## Scenario Sequence

```mermaid
sequenceDiagram
    participant CLI
    participant RDLoop
    participant Developer
    participant Workspace
    participant KnowledgeBase
    participant Evaluator

    CLI->>RDLoop: rdagent fin_factor
    activate RDLoop
    
    RDLoop->>RDLoop: _propose()
    Note over RDLoop: Generate factor hypothesis
    
    RDLoop->>RDLoop: _exp_gen()
    Note over RDLoop: Create experiment from hypothesis
    
    RDLoop->>Developer: develop(experiment)
    activate Developer
    Developer->>Workspace: prepare()
    Developer->>Workspace: inject_code()
    Developer->>Workspace: execute()
    Developer-->>RDLoop: return experiment
    deactivate Developer
    
    RDLoop->>Evaluator: evaluate()
    activate Evaluator
    Evaluator->>KnowledgeBase: store results
    Evaluator-->>RDLoop: return feedback
    deactivate Evaluator
    
    RDLoop-->>CLI: Complete factor development
    deactivate RDLoop
```

## Use of LiteLLM

```mermaid
sequenceDiagram
    participant CLI
    participant RDLoop
    participant LiteLLMAPI
    participant Developer
    participant KnowledgeBase
    
    CLI->>RDLoop: rdagent fin_factor
    activate RDLoop
    
    RDLoop->>LiteLLMAPI: create_chat_completion(propose_prompt)
    Note over LiteLLMAPI: Using model from settings<br/>Temperature and tokens configured
    activate LiteLLMAPI
    LiteLLMAPI-->>RDLoop: Factor hypothesis
    deactivate LiteLLMAPI
    
    RDLoop->>KnowledgeBase: query previous factors
    KnowledgeBase-->>RDLoop: related factors
    
    RDLoop->>LiteLLMAPI: create_embedding(hypothesis)
    activate LiteLLMAPI
    LiteLLMAPI-->>RDLoop: hypothesis embedding
    deactivate LiteLLMAPI
    
    RDLoop->>LiteLLMAPI: create_chat_completion(code_gen_prompt)
    Note over LiteLLMAPI: Code generation with<br/>context from knowledge base
    activate LiteLLMAPI
    LiteLLMAPI-->>RDLoop: Factor implementation code
    deactivate LiteLLMAPI
    
    RDLoop->>Developer: develop(experiment)
    activate Developer
    Developer->>LiteLLMAPI: create_chat_completion(debug_prompt)
    activate LiteLLMAPI
    Note over LiteLLMAPI: Debugging assistance<br/>with different model settings
    LiteLLMAPI-->>Developer: Debug suggestions
    deactivate LiteLLMAPI
    Developer-->>RDLoop: Completed experiment
    deactivate Developer
    
    RDLoop->>LiteLLMAPI: create_chat_completion(evaluate_prompt)
    activate LiteLLMAPI
    Note over LiteLLMAPI: Evaluation with<br/>specific reasoning effort
    LiteLLMAPI-->>RDLoop: Evaluation results
    deactivate LiteLLMAPI
    
    RDLoop->>KnowledgeBase: store_results()
    RDLoop-->>CLI: Complete factor development
    deactivate RDLoop
```

The sequence diagram shows how LiteLLM integrates with the factor development process:

1. Initial Hypothesis Generation:
   - Uses chat model with controlled temperature for creative yet focused proposals
   - Prompts include previous research context and requirements

2. Code Generation:
   - Higher token limit for complex code generation
   - Prompts include knowledge base context and specific factor requirements
   - Uses JSON mode for structured output when needed

3. Debugging Support:
   - Different model configuration for debugging (specified in chat_model_map)
   - Focused temperature settings for precise problem-solving

4. Evaluation Phase:
   - Uses specific reasoning_effort setting for thorough analysis
   - Structured prompts for consistent evaluation criteria

Throughout the process, LiteLLM handles:
- Token counting and cost tracking
- Model-specific configurations
- Streaming responses for real-time feedback
- Embedding generation for knowledge base integration
