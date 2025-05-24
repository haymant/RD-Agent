# Factor Workflow Sequence Diagram

The following diagram illustrates the sequence of operations triggered by the factor.py script in the RD-Agent system.

```mermaid
sequenceDiagram
    participant User
    participant Main as factor.py:main()
    participant FactorRDLoop
    participant RDLoop
    participant Runner
    participant Logger

    User->>Main: Run script with path/step_n
    
    alt No path provided
        Main->>FactorRDLoop: Create new(FACTOR_PROP_SETTING)
    else Path provided
        Main->>FactorRDLoop: load(path)
    end
    
    Main->>FactorRDLoop: run(step_n)
    FactorRDLoop->>RDLoop: run(step_n)
    
    loop For each step
        Note over RDLoop: Get previous output
        
        alt First step or propose step
            RDLoop->>RDLoop: propose()
            Note over RDLoop: Generate new ideas
        else if coding step
            RDLoop->>RDLoop: coding()
            Note over RDLoop: Implement code
        else if running step
            RDLoop->>FactorRDLoop: running(prev_out)
            FactorRDLoop->>Logger: tag("ef")
            FactorRDLoop->>Runner: develop(prev_out["coding"])
            
            alt Factor extraction successful
                Runner-->>FactorRDLoop: Return experiment
                FactorRDLoop->>Logger: log_object(exp)
            else Factor extraction fails
                Runner-->>FactorRDLoop: Return None
                FactorRDLoop->>Logger: error("Factor extraction failed")
                FactorRDLoop->>FactorRDLoop: raise FactorEmptyError
                Note over FactorRDLoop, RDLoop: Error caught by skip_loop_error
            end
        end
        
        RDLoop->>RDLoop: Save session state
    end
    
    FactorRDLoop-->>Main: Return results
    Main-->>User: Execution complete
```

This diagram shows the full lifecycle of a factor development iteration, including:
1. Script initialization with optional loading of existing sessions
2. Execution of the R&D loop
3. The specific implementation of the `running()` method in FactorRDLoop
4. Error handling for factor extraction failures
5. Session persistence between iterations
