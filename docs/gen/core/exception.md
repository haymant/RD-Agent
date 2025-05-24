# Exception Handling

## Overview
The RD-Agent system uses specialized exceptions to manage errors in the research and development workflow. These exceptions allow for graceful handling of expected failure cases.

## Key Exception Classes

### FactorEmptyError
Raised when factor extraction fails to produce a valid result, commonly during the running phase.

#### Usage
```python
from rdagent.core.exception import FactorEmptyError

try:
    # Factor extraction code
    if factor_is_empty:
        raise FactorEmptyError("Factor extraction failed")
except FactorEmptyError as e:
    # Handle gracefully
    logger.error(f"Error in factor extraction: {e}")
```

### Workflow Integration
Exceptions are integrated with the RDLoop workflow through the `skip_loop_error` attribute, which defines which errors should cause the loop to skip an iteration rather than halt execution.

Example:
```python
class CustomRDLoop(RDLoop):
    skip_loop_error = (FactorEmptyError, OtherCustomError)
    
    def running(self, prev_out):
        try:
            # Implementation
        except FactorEmptyError:
            # Will be caught by the RDLoop and handled according to policy
            raise
```

### Benefits
- **Resilience**: Workflows continue despite non-fatal errors
- **Clarity**: Error conditions are explicitly defined and recognizable
- **Separation of Concerns**: Error handling policy is defined separately from implementation details
