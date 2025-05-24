# RD Loop Component

## Overview
The RDLoop (Research & Development Loop) is the core workflow engine in the RD-Agent system. It orchestrates the iterative process of proposing, implementing, testing, and refining models or factors.

## Class: RDLoop

### Purpose
Provides a structured framework for iterative research and development workflows, with built-in persistence, logging, and error handling.

### Key Features
- **Session Management**: Maintains state between iterations and allows resuming from checkpoints
- **Configurable Workflow**: Supports customizable stages in the R&D process
- **Error Handling**: Gracefully manages expected errors with `skip_loop_error`
- **Persistence**: Saves progress at each step for analysis and continuation

### Methods
- **run()**: Executes the full R&D loop for a specified number of iterations
- **load()**: Restores a workflow from a previously saved state
- **running()**: Implements the evaluation phase of the workflow

### Extension Points
The RDLoop is designed to be extended by specific implementations like FactorRDLoop, which can:
- Define domain-specific error handling
- Implement specialized running and evaluation logic
- Provide domain-appropriate configuration

### Usage Example
```python
from rdagent.components.workflow.rd_loop import RDLoop

# Create a new instance with configuration
loop = RDLoop(config_settings)

# Run the loop
loop.run(step_n=5)

# Resume a previous session
resumed_loop = RDLoop.load("/path/to/session")
resumed_loop.run()
```
