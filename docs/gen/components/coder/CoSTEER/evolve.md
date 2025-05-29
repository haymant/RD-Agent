# CoSTEER Evolution System

## Overview

The CoSTEER (Component-based Strategy for Test-driven Evolution and Evaluation Refinement) evolution system is a framework for iteratively improving AI-generated code implementations through a structured evolution process. It consists of three main components:

1. **Evolvable Subjects**: Entities that can evolve over time
2. **Evolving Strategies**: Algorithms that drive the evolution process
3. **Evaluators**: Systems that assess the quality of evolved implementations

This document explains how these components work together to enable continuous improvement of code implementations.

## Evolvable Subjects

### EvolvingItem

The `EvolvingItem` class is the core evolvable subject in the CoSTEER system. It represents an intermediate representation of factor implementation that can evolve over time.

```python
class EvolvingItem(Experiment, EvolvableSubjects):
    """
    Intermediate item of factor implementation.
    """
```

Key features:
- **Multiple Inheritance**: Inherits from both `Experiment` and `EvolvableSubjects`
- **Task Management**: Contains a list of tasks to be implemented
- **Ground Truth Support**: Optionally stores ground truth implementations for evaluation
- **Experiment Conversion**: Can be created from an existing experiment via the `from_experiment` method

The EvolvingItem serves as the central subject that evolves through the iterative process, with its implementations being refined at each step.

## Evolving Strategies

### MultiProcessEvolvingStrategy

The `MultiProcessEvolvingStrategy` class implements the core evolution algorithm that transforms an evolvable subject through multiple iterations.

Key components:

1. **Task Selection**: Identifies tasks that need evolution based on their current state and knowledge base
   ```python
   to_be_finished_task_index: list[int] = []
   for index, target_task in enumerate(evo.sub_tasks):
       target_task_desc = target_task.get_task_information()
       if target_task_desc in queried_knowledge.success_task_to_knowledge_dict:
           # Use existing knowledge for already successful tasks
       elif target_task_desc not in queried_knowledge.failed_task_info_set:
           # Add to tasks needing implementation
           to_be_finished_task_index.append(index)
   ```

2. **Parallel Implementation**: Uses multiprocessing to implement multiple tasks concurrently
   ```python
   result = multiprocessing_wrapper(
       [
           (
               self.implement_one_task,
               (
                   evo.sub_tasks[target_index],
                   queried_knowledge,
                   evo.experiment_workspace,
                   None if last_feedback is None else last_feedback[target_index],
               ),
           )
           for target_index in to_be_finished_task_index
       ],
       n=RD_AGENT_SETTINGS.multi_proc_n,
   )
   ```

3. **Abstract Methods**: Defines two key abstract methods that concrete implementations must provide:
   - `implement_one_task`: Implements a single task with potential knowledge and feedback
   - `assign_code_list_to_evo`: Applies implementation results to the evolving item

The strategy uses feedback from previous iterations and knowledge from RAG to intelligently evolve the implementations.

## Evaluators

The evaluation system in CoSTEER provides structured feedback on implementations, helping the evolution process learn from successes and failures.

### Feedback Structure

#### CoSTEERSingleFeedback

This class provides comprehensive feedback on a single implementation:

```python
@dataclass
class CoSTEERSingleFeedback(Feedback):
    execution: str                # Feedback on code execution
    return_checking: str | None   # Validation of return values
    code: str                     # Feedback on code quality
    final_decision: bool          # Overall success/failure
```

The feedback follows a logical flow:
1. **Execution**: Did the code run without errors?
2. **Return Checking**: Did the code produce correct outputs?
3. **Code Quality**: Is the code well-written and efficient?
4. **Final Decision**: Overall determination of success or failure

#### CoSTEERMultiFeedback

A container for multiple `CoSTEERSingleFeedback` instances, providing list-like behavior:

```python
class CoSTEERMultiFeedback(Feedback):
    """Feedback contains a list, each element is the corresponding feedback for each factor implementation."""
```

Key features:
- List-like behavior with indexing, iteration, and length methods
- Methods to determine if all tasks are successfully completed
- Handles cases where some tasks may have been skipped (resulting in `None` feedback)

### Evaluator Classes

#### CoSTEEREvaluator

An abstract base class for evaluating a single task implementation:

```python
class CoSTEEREvaluator(Evaluator):
    @abstractmethod
    def evaluate(
        self,
        target_task: Task,
        implementation: Workspace,
        gt_implementation: Workspace,
        **kwargs,
    ) -> CoSTEERSingleFeedback:
        raise NotImplementedError("Please implement the `evaluator` method")
```

#### CoSTEERMultiEvaluator

Evaluates multiple tasks, possibly using multiple evaluators:

```python
class CoSTEERMultiEvaluator(CoSTEEREvaluator):
    """This is for evaluation of experiment. Due to we have multiple tasks, so we will return a list of evaluation feedbacks"""
```

Key features:
- Supports a single evaluator or a list of evaluators
- Uses multiprocessing for parallel evaluation
- Merges feedback from multiple evaluators
- Updates task status based on evaluation results

## Evolution Process Flow

The complete evolution process in CoSTEER follows these steps:

1. **Initialization**: An `EvolvingItem` is created with tasks to be implemented
2. **Querying Knowledge**: The system retrieves relevant knowledge from the knowledge base
3. **Evolution**:
   - Tasks are classified as completed, failed, or needing implementation
   - Tasks needing implementation are processed in parallel
   - New implementations are applied to the evolving item
4. **Evaluation**:
   - Each implementation is evaluated against defined criteria
   - Comprehensive feedback is generated for each task
   - The system determines which tasks succeeded and which need further evolution
5. **Iteration**: Steps 2-4 repeat until all tasks succeed or reach the maximum iteration limit

## Integration with RAG

The evolution process integrates with Retrieval-Augmented Generation (RAG) through:

1. **Knowledge Retrieval**: Using `queried_knowledge` to inform implementation decisions
2. **Feedback History**: Accessing previous feedback through `evolving_trace`
3. **Continuous Learning**: Updating the knowledge base with successful implementations

This integration allows the system to learn from past successes and failures, making each evolution step more informed than the last.
