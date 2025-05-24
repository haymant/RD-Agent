# RD-Agent: Research & Development Agent

## Project Overview
RD-Agent is an automated research and development framework designed for financial technology applications, particularly focused on factor development in quantitative finance. The system utilizes an iterative loop approach to continuously refine and evaluate financial factors.

## Architecture

The RD-Agent architecture follows a modular design with several key components:

### Core Components
- **RDLoop**: Base workflow engine that manages the research and development iteration process
- **Runner**: Execution engine that handles the development and evaluation of models
- **Exception Handling**: Specialized error types for graceful failure management
- **EvolvingAgent**: Framework for implementing AI that evolves through multiple iterations with RAG

### Main Modules
- **rdagent.app**: Contains application-specific implementations
  - **qlib_rd_loop**: Specialized for quantitative library factor development
- **rdagent.components**: Reusable workflow components
  - **workflow**: Contains the RD loop implementation
  - **coder**: Implementations of AI code generation strategies
- **rdagent.core**: Core functionality and base classes

## Workflow

The system follows a Research & Development loop pattern:

1. **Proposal**: Generate new factor ideas or improvements
2. **Coding**: Implement the proposed factors
3. **Running**: Execute and evaluate the implemented factors
4. **Feedback**: Analyze results and generate insights for the next iteration

### FactorRDLoop

The `FactorRDLoop` class extends the base `RDLoop` system for factor development by:
- Managing factor extraction and evaluation
- Handling factor-specific errors gracefully
- Providing session persistence for continuing experiments

## Getting Started

To run a new factor development session:
```python
python rdagent/app/qlib_rd_loop/factor.py
```

To continue an existing session:
```python
python rdagent/app/qlib_rd_loop/factor.py $LOG_PATH/__session__/1/0_propose --step_n 1
```

## Component Documentation

For detailed information about specific components, please refer to the following documentation:
- [RDLoop Documentation](./components/rd_loop.md)
- [Runner Documentation](./core/runner.md)
- [Exception Handling](./core/exception.md)
- [Evolving AI Framework](./core/evolve.md)
