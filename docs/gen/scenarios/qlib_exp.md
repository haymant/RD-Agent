# Qlib Experiment Components

## Overview

The experiment directory contains the core components that enable the RD-Agent to integrate with Qlib, Microsoft's AI-oriented quantitative investment platform. This directory houses the essential classes, configurations, and templates needed to execute factor and model experiments within the quantitative finance domain.

## Directory Structure



## Core Components

### Workspace Management

#### `QlibFBWorkspace` (workspace.py)

This specialized workspace class provides the execution environment for both factor and model experiments:

- **Template Injection**: Copies template files to new experiment workspace
- **Docker Execution**: Runs Qlib in isolated containers
- **Result Processing**: Retrieves and parses backtest results

```python
def execute(self, qlib_config_name: str = "conf.yaml", run_env: dict = {}, *args, **kwargs) -> str:
    qtde = QTDockerEnv()
    qtde.prepare()

    # Run the Qlib backtest
    execute_log = qtde.run(
        local_path=str(self.workspace_path),
        entry=f"qrun {qlib_config_name}",
        env=run_env,
    )
    
    # Read experiment results and return metrics
    # ...existing code...
```

The workspace handles all interaction with Qlib's execution environment, making the experiment process reproducible and isolated.

### Utility Functions

#### Data Management (utils.py)

The utility module provides essential functions for data preparation and interpretation:

- **`generate_data_folder_from_qlib()`**: Creates standardized dataset directories
- **`get_file_desc()`**: Generates human-readable data file descriptions
- **`get_data_folder_intro()`**: Prepares documentation for available datasets

These utilities bridge the gap between Qlib's data structures and RD-Agent's prompting system, making financial data accessible to language models.

### Factor Experiments

#### `QlibFactorExperiment` (factor_experiment.py)

Extends the base `FactorExperiment` with Qlib-specific functionality:

```python
def __init__(self, *args, **kwargs) -> None:
    super().__init__(*args, **kwargs)
    self.experiment_workspace = QlibFBWorkspace(template_folder_path=Path(__file__).parent / "factor_template")
```

This class initializes experiment workspaces with the necessary templates and configurations for financial factor evaluation.

#### `QlibFactorScenario` (factor_experiment.py)

Provides the domain context for factor development through a collection of prompts:

```python
def get_scenario_all_desc(self, task: Task | None = None, filtered_tag: str | None = None, simple_background: bool | None = None) -> str:
    """A static scenario describer"""
    # ...existing code...
    return f"""Background of the scenario:
{self.background}
The source data you can use:
{self.get_source_data_desc(task)}
The interface you should follow to write the runnable code:
{self.interface}
The output of your code should be in the format:
{self.output_format}
The simulator user can use to test your factor:
{self.simulator}
"""
```

This provides a complete context document that guides the LLM in generating appropriate factor implementations.

### Model Experiments

#### `QlibModelExperiment` (model_experiment.py)

Similar to its factor counterpart, this class manages model-specific implementations:

```python
def __init__(self, *args, **kwargs) -> None:
    super().__init__(*args, **kwargs)
    self.experiment_workspace = QlibFBWorkspace(template_folder_path=Path(__file__).parent / "model_template")
```

It initializes workspaces with PyTorch model templates and Qlib configurations.

#### `QlibModelScenario` (model_experiment.py)

Provides machine learning context for financial model development:

```python
def get_scenario_all_desc(self, task: Task | None = None, filtered_tag: str | None = None, simple_background: bool | None = None) -> str:
    return f"""Background of the scenario:
{self.background}
The interface you should follow to write the runnable code:
{self.interface}
The output of your code should be in the format:
{self.output_format}
The simulator user can use to test your model:
{self.simulator}
"""
```

This scenario guides the LLM in implementing appropriate ML models for financial forecasting.

### Report-Based Factor Extraction

#### `QlibFactorFromReportScenario` (factor_from_report_experiment.py)

Extends the base factor scenario with specialized context for extracting factors from research papers:

```python
class QlibFactorFromReportScenario(QlibFactorScenario):
    def __init__(self) -> None:
        super().__init__()
        self._rich_style_description = deepcopy(prompt_dict["qlib_factor_from_report_rich_style_description"])
```

The accompanying PDF loader component (`pdf_loader.py`) implements an advanced pipeline for extracting mathematical formulations from research documents.

### Configuration Files

#### Factor Configurations

The experiment templates include several YAML configurations:

- **`conf.yaml`**: Standard configuration for single factor experiments
  ```yaml
  task:
      model:
          class: LGBModel
          # ...existing code...
      dataset:
          class: DatasetH
          # ...existing code...
  ```

- **`conf_combined.yaml`**: Enhanced configuration that supports combined factor evaluation
  ```yaml
  data_handler_config:
      # ...existing code...
      data_loader:
          class: NestedDataLoader
          kwargs:
              dataloader_l:
                  - class: qlib.contrib.data.loader.Alpha158DL
                  # ...existing code...
                  - class: qlib.data.dataset.loader.StaticDataLoader
  ```

#### Model Configuration

The model template includes a configuration optimized for PyTorch models:

```yaml
task:
    model:
        class: GeneralPTNN
        module_path: qlib.contrib.model.pytorch_general_nn
        kwargs:
            # ...existing code...
            pt_model_uri: "model.model_cls"
            pt_model_kwargs: {
                "num_features": 20,
                {% if num_timesteps %}num_timesteps: {{ num_timesteps }}{% endif %}
            }
```

This configuration uses Jinja templating to dynamically adapt to different model types.

### Result Processing

Both factor and model templates include a `read_exp_res.py` script that:

1. Connects to Qlib's experiment tracking system
2. Locates the latest recorder with completed results
3. Extracts performance metrics and stores them in standardized formats

```python
# Find the latest recorder
# ...existing code...

# Extract metrics and save
metrics = pd.Series(latest_recorder.list_metrics())
output_path = Path(__file__).resolve().parent / "qlib_res.csv"
metrics.to_csv(output_path)

# Retrieve portfolio analysis results for visualization
ret_data_frame = latest_recorder.load_object("portfolio_analysis/report_normal_1day.pkl")
ret_data_frame.to_pickle("ret.pkl")
```

### Data Templates

The `factor_data_template` directory contains resources for consistent data handling:

- **README.md**: Documentation for data structure and access patterns
  ```markdown
  ## Daily price and volume data
  $open: open price of the stock on that day.
  $close: close price of the stock on that day.
  # ...existing code...
  ```

- **generate.py**: Script that produces standardized data snapshots
  ```python
  fields = ["$open", "$close", "$high", "$low", "$volume", "$factor"]
  data = D.features(instruments, fields, freq="day").swaplevel().sort_index()
  # ...existing code...
  ```

These templates ensure consistent data formatting across all experiments.

### Integration with RD-Agent

The experiment components connect seamlessly with RD-Agent's workflow:

1. **Proposal System**: `factor_proposal.py` and `model_proposal.py` generate hypotheses
2. **Experiment Loaders**: JSON and PDF loaders prepare tasks from various sources
3. **Developers**: `factor_runner.py` and `model_runner.py` execute implementations
4. **Feedback**: `feedback.py` evaluates results and guides future iterations

The `prompts.yaml` file serves as the coordination layer, providing templates that enable language models to interact with each component in a domain-appropriate manner.