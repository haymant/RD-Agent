# Knowledge Management in CoSTEER

## Overview

The knowledge management system in CoSTEER (Component-based Strategy for Test-driven Evolution and Evaluation Refinement) provides an intelligent memory mechanism for the evolutionary AI framework. It implements Retrieval-Augmented Generation (RAG) strategies to capture, store, retrieve, and utilize knowledge gained through multiple iterations of development.

## Knowledge Representation

### Core Knowledge Classes

- **CoSTEERKnowledge**: The fundamental unit of knowledge, containing:
  - A target task description
  - An implementation (code workspace)
  - Feedback about the implementation
  
- **CoSTEERQueriedKnowledge**: Container for knowledge retrieved during the RAG process, including:
  - Successful task implementations
  - Failed task information
  - Former traces of implementations
  - Similar task knowledge
  - Error-related knowledge

## Knowledge Base Evolution

The system supports two versions of knowledge bases with increasing sophistication:

### CoSTEERKnowledgeBaseV1

A simple implementation focused on storing and retrieving successful implementations:

- Maintains a dictionary of implementation traces indexed by task description
- Tracks successful implementations in a separate set
- Supports simple similarity-based retrieval

### CoSTEERKnowledgeBaseV2

An advanced implementation that uses a graph-based knowledge representation:

- Stores knowledge in an undirected graph structure
- Organizes knowledge nodes by type (component, task_description, task_trace, task_success_implement, error)
- Supports sophisticated querying based on node relationships
- Enables analysis of components and errors for more effective knowledge retrieval

## RAG Strategies

The system implements two RAG (Retrieval-Augmented Generation) strategies:

### CoSTEERRAGStrategyV1

Basic implementation with limited functionality:

- Retrieves similar successful implementations based on embedding distance
- Tracks failed attempts to avoid repetition
- Limited knowledge generation capabilities

### CoSTEERRAGStrategyV2

Advanced implementation with comprehensive knowledge management:

- **Knowledge Generation**: Analyzes the evolution trace to create new knowledge entries
- **Component Analysis**: Identifies relevant components for a task
- **Error Analysis**: Extracts and categorizes errors for future reference
- **Multiple Query Types**:
  - **Former Trace Query**: Retrieves previous attempts for the same task
  - **Component Query**: Finds implementations with similar components
  - **Error Query**: Locates implementations that encountered and solved similar errors

## Graph-Based Knowledge Representation

CoSTEERKnowledgeBaseV2 uses a graph structure where:

- **Nodes** represent different types of information:
  - Component nodes (reusable code components)
  - Task description nodes
  - Task trace nodes (implementation attempts)
  - Task success implementation nodes
  - Error nodes
  
- **Edges** represent relationships between nodes:
  - Task descriptions link to relevant components
  - Traces link to errors encountered
  - Success implementations link to their task descriptions
  - Components link to related task descriptions

## Query Mechanisms

### Former Trace Query

Retrieves previous attempts for the current task:

```python
queried_knowledge_v2 = self.former_trace_query(
    evo,
    queried_knowledge_v2,
    self.settings.v2_query_former_trace_limit,
    self.settings.v2_add_fail_attempt_to_latest_successful_execution,
)
```

- Identifies tasks that have been attempted too many times
- Retrieves the most recent attempts, avoiding deteriorating trials
- Optionally includes the latest attempt after a successful execution

### Component Query

Identifies and retrieves knowledge based on shared components:

```python
queried_knowledge_v2 = self.component_query(
    evo,
    queried_knowledge_v2,
    self.settings.v2_query_component_limit,
    knowledge_sampler=conf_knowledge_sampler,
)
```

- Analyzes tasks to identify relevant components
- Queries the knowledge graph for tasks using the same components
- Uses intersection search to find tasks using multiple components
- Balances knowledge from ground truth and other sources

### Error Query

Retrieves implementations that encountered and solved similar errors:

```python
queried_knowledge_v2 = self.error_query(
    evo,
    queried_knowledge_v2,
    self.settings.v2_query_error_limit,
    knowledge_sampler=conf_knowledge_sampler,
)
```

- Extracts error information from previous attempts
- Locates task traces that encountered similar errors
- Finds successful implementations that addressed those errors
- Returns pairs of (error trace, successful implementation)

## Knowledge Generation and Update

When a task is successfully implemented, the system:

1. Creates a task description node in the graph
2. Links it to relevant component nodes
3. Creates nodes for each implementation attempt (traces)
4. Links traces to encountered errors
5. Creates a special node for the successful implementation
6. Updates the success_task_to_knowledge_dict

## Graph Query Mechanisms

The knowledge base supports sophisticated graph queries:

- **Content-based queries**: Find nodes with similar content
- **Node-based queries**: Find connected nodes within a certain number of steps
- **Intersection queries**: Find nodes connected to multiple starting nodes
- **Constrained queries**: Limit results by node type, connection distance, etc.

## Knowledge Sampling

To manage computational resources, the system supports knowledge sampling:

```python
if knowledge_sampler > 0:
    queried_knowledge_v2.task_to_similar_task_successful_knowledge[target_task_information] = [
        knowledge
        for knowledge in queried_knowledge_v2.task_to_similar_task_successful_knowledge[target_task_information]
        if random.uniform(0, 1) <= knowledge_sampler
    ]
```

This allows controlling the amount of knowledge retrieved for each query.
