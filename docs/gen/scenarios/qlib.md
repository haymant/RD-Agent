# Qlib Integration in RD-Agent

## Overview

Qlib is an AI-oriented quantitative investment platform integrated into RD-Agent to enable automated research and development in financial factor modeling and trading strategies. This document explains how the various components work together to form a comprehensive R&D workflow for quantitative finance.

## Prompt-Based Workflow Coordination

The RD-Agent system uses prompt engineering extensively to coordinate different phases of the research and development process. The central component enabling this coordination is the `prompts.yaml` file.

### Prompts Structure

The prompts.yaml file contains structured templates for different aspects of the workflow:

1. **Hypothesis Generation**
   - `hypothesis_and_feedback`: Template for presenting previous hypotheses and their feedback
   - `hypothesis_output_format`: JSON schema for hypothesis output
   - `factor_hypothesis_specification`: Domain-specific guidelines for factor hypotheses
   - `model_hypothesis_specification`: Domain-specific guidelines for model architecture hypotheses

2. **Experiment Generation**
   - `factor_experiment_output_format`: JSON schema for factor experiments
   - `model_experiment_output_format`: JSON schema for model experiments

3. **Feedback Generation**
   - `factor_feedback_generation`: System and user prompts for generating feedback on factor performance
   - `model_feedback_generation`: System and user prompts for assessing model architecture performance

4. **Domain Knowledge**
   - `qlib_factor_background`: Background information on financial factors
   - `qlib_factor_interface`: Technical interface requirements for factor implementation
   - `qlib_model_background`: Background information on machine learning models
   - `qlib_model_interface`: Technical interface requirements for model implementation

### Workflow Coordination

The prompts facilitate a coordinated workflow across the following stages:

1. **Hypothesis Generation**: Prompts guide the creation of testable hypotheses about financial factors or model architectures
2. **Experiment Creation**: Hypotheses are transformed into concrete experiment specifications
3. **Implementation**: Code is generated based on experiment specifications
4. **Evaluation**: Results are analyzed and feedback is generated
5. **Iteration**: Feedback informs the next round of hypothesis generation

This prompt-based approach allows for a flexible, language-model-driven research process that can adapt to different scenarios and problem domains.

## Proposal System

The proposal system is responsible for generating hypotheses and converting them into executable experiments.

### Factor Proposal

The factor proposal system is implemented in `factor_proposal.py` and consists of two main classes:

#### QlibFactorHypothesisGen

Generates hypotheses for financial factors based on previous experiments:

```python
def prepare_context(self, trace: Trace) -> Tuple[dict, bool]:
    hypothesis_and_feedback = (
        (
            Environment(undefined=StrictUndefined)
            .from_string(prompt_dict["hypothesis_and_feedback"])
            .render(trace=trace)
        )
        if len(trace.hist) > 0
        else "No previous hypothesis and feedback available since it's the first round."
    )
    context_dict = {
        "hypothesis_and_feedback": hypothesis_and_feedback,
        "RAG": None,
        "hypothesis_output_format": prompt_dict["hypothesis_output_format"],
        "hypothesis_specification": prompt_dict["factor_hypothesis_specification"],
    }
    return context_dict, True
```

This class:
- Renders the history of previous hypotheses and their feedback
- Provides format specifications and domain-specific guidelines
- Converts API responses into structured `QlibFactorHypothesis` objects

#### QlibFactorHypothesis2Experiment

Converts hypotheses into executable factor experiments:

```python
def convert_response(self, response: str, hypothesis: Hypothesis, trace: Trace) -> FactorExperiment:
    response_dict = json.loads(response)
    tasks = []

    for factor_name in response_dict:
        description = response_dict[factor_name]["description"]
        formulation = response_dict[factor_name]["formulation"]
        variables = response_dict[factor_name]["variables"]
        tasks.append(
            FactorTask(
                factor_name=factor_name,
                factor_description=description,
                factor_formulation=formulation,
                variables=variables,
            )
        )

        exp = QlibFactorExperiment(tasks, hypothesis=hypothesis)
        exp.based_experiments = [QlibFactorExperiment(sub_tasks=[])] + [t[0] for t in trace.hist if t[1]]

        unique_tasks = []

        for task in tasks:
            duplicate = False
            for based_exp in exp.based_experiments:
                for sub_task in based_exp.sub_tasks:
                    if task.factor_name == sub_task.factor_name:
                        duplicate = True
                        break
                if duplicate:
                    break
            if not duplicate:
                unique_tasks.append(task)

        exp.tasks = unique_tasks
        return exp
```

This class:
- Takes a hypothesis as input
- Prepares context with the hypothesis and experiment history
- Converts API responses into `QlibFactorExperiment` objects
- Ensures experiments include only unique tasks

### Model Proposal

The model proposal system is implemented in `model_proposal.py` and follows a similar structure:

#### QlibModelHypothesisGen

Generates hypotheses for model architectures:

```python
def prepare_context(self, trace: Trace) -> Tuple[dict, bool]:
    # ...existing code...
    context_dict = {
        "hypothesis_and_feedback": hypothesis_and_feedback,
        "RAG": "In Quantitative Finance, market data could be time-series, and GRU model/LSTM model are suitable for them. Do not generate GNN model as for now.",
        "hypothesis_output_format": prompt_dict["hypothesis_output_format"],
        "hypothesis_specification": prompt_dict["model_hypothesis_specification"],
    }
    return context_dict, True
```

This class adds domain-specific RAG content to guide hypothesis generation for time-series models.

#### QlibModelHypothesis2Experiment

Converts model hypotheses into executable experiments:

```python
def convert_response(self, response: str, hypothesis: Hypothesis, trace: Trace) -> ModelExperiment:
    response_dict = json.loads(response)
    tasks = []
    for model_name in response_dict:
        description = response_dict[model_name]["description"]
        formulation = response_dict[model_name]["formulation"]
        architecture = response_dict[model_name]["architecture"]
        variables = response_dict[model_name]["variables"]
        hyperparameters = response_dict[model_name]["hyperparameters"]
        model_type = response_dict[model_name]["model_type"]
        tasks.append(
            ModelTask(
                name=model_name,
                description=description,
                formulation=formulation,
                architecture=architecture,
                variables=variables,
                hyperparameters=hyperparameters,
                model_type=model_type,
            )
        )
        exp = QlibModelExperiment(tasks, hypothesis=hypothesis)
        exp.based_experiments = [t[0] for t in trace.hist if t[1]]
        return exp
```

The model experiment generation includes additional fields for architecture and hyperparameters.

## Factor Experiment Loader

The Factor Experiment Loader components are responsible for loading factor definitions from various sources.

### JSON Loader

The JSON loader (`json_loader.py`) provides classes for loading factor experiments from JSON data:

#### FactorExperimentLoaderFromDict

Loads factor experiments from a Python dictionary:

```python
def load(self, factor_dict: dict) -> list:
    """Load data from a dict."""
    task_l = []
    for factor_name, factor_data in factor_dict.items():
        task = FactorTask(
            factor_name=factor_name,
            factor_description=factor_data["description"],
            factor_formulation=factor_data["formulation"],
            variables=factor_data["variables"],
        )
        task_l.append(task)
    exp = QlibFactorExperiment(sub_tasks=task_l)
    return exp
```

#### FactorExperimentLoaderFromJsonFile and FactorExperimentLoaderFromJsonString

These classes extend the dict loader to handle file and string inputs respectively.

#### FactorTestCaseLoaderFromJsonFile

Loads test cases for factors, including ground truth implementations:

```python
def load(self, json_file_path: Path) -> TestCases:
    # ...existing code...
    for factor_name, factor_data in factor_dict.items():
        task = FactorTask(
            factor_name=factor_name,
            factor_description=factor_data["description"],
            factor_formulation=factor_data["formulation"],
            variables=factor_data["variables"],
        )
        gt = FactorFBWorkspace(task, raise_exception=False)
        code = {"factor.py": factor_data["gt_code"]}
        gt.inject_files(**code)
        test_cases.test_case_l.append(TestCase(task, gt))

    return test_cases
```

### PDF Loader

The PDF loader (`pdf_loader.py`) provides sophisticated capabilities for extracting factor definitions from research papers:

#### FactorExperimentLoaderFromPDFfiles

This class orchestrates the end-to-end process of extracting factors from PDF files:

```python
def load(self, file_or_folder_path: str) -> dict:
    with logger.tag("docs"):
        docs_dict = load_and_process_pdfs_by_langchain(file_or_folder_path)
        logger.log_object(docs_dict)

    selected_report_dict = classify_report_from_dict(report_dict=docs_dict, vote_time=1)

    with logger.tag("file_to_factor_result"):
        file_to_factor_result = extract_factors_from_report_dict(docs_dict, selected_report_dict)
        logger.log_object(file_to_factor_result)

    with logger.tag("factor_dict"):
        factor_dict = merge_file_to_factor_dict_to_factor_dict(file_to_factor_result)
        logger.log_object(factor_dict)

    with logger.tag("filtered_factor_dict"):
        factor_viability, filtered_factor_dict = check_factor_viability(factor_dict)
        logger.log_object(filtered_factor_dict)

    # factor_dict, duplication_names_list = deduplicate_factors_by_llm(factor_dict, factor_viability)

    return FactorExperimentLoaderFromDict().load(filtered_factor_dict)
```

The PDF loading process involves several steps:

1. **Document Loading**: Using LangChain to load and process PDF files
2. **Report Classification**: Identifying relevant reports using an LLM-based classifier
3. **Factor Extraction**: Extracting factor names, descriptions, and formulations
4. **Formulation Parsing**: Processing mathematical formulations and variables
5. **Deduplication**: Identifying and merging duplicate factors
6. **Validation**: Checking factor viability and relevance

Key helper functions include:
- `extract_factors_name_and_desc_from_content`: Extracts factor names and descriptions
- `extract_factors_formulation_from_content`: Extracts mathematical formulations and variables
- `check_factor_viability`: Validates if factors are implementable
- `deduplicate_factors_by_llm`: Uses LLM and embedding similarity to identify duplicates

This sophisticated extraction pipeline allows the RD-Agent to incorporate knowledge from academic literature and research reports into its factor development process.

## Developer Components

The Developer components are responsible for implementing and evaluating experiments.

### Factor Development

#### QlibFactorRunner

The `QlibFactorRunner` class in `factor_runner.py` handles the execution of factor experiments:

```python
def develop(self, exp: QlibFactorExperiment) -> QlibFactorExperiment:
    """
    Generate the experiment by processing and combining factor data,
    then passing the combined data to Docker for backtest results.
    """
    if exp.based_experiments and exp.based_experiments[-1].result is None:
        exp.based_experiments[-1] = self.develop(exp.based_experiments[-1])

    if exp.based_experiments:
        SOTA_factor = None
        if len(exp.based_experiments) > 1:
            SOTA_factor = self.process_factor_data(exp.based_experiments)

        # Process the new factors data
        new_factors = self.process_factor_data(exp)

        if new_factors.empty:
            raise FactorEmptyError("No valid factor data found to merge.")

        # Combine the SOTA factor and new factors if SOTA factor exists
        if SOTA_factor is not None and not SOTA_factor.empty:
            new_factors = self.deduplicate_new_factors(SOTA_factor, new_factors)
            if new_factors.empty:
                raise FactorEmptyError("No valid factor data found to merge.")
            combined_factors = pd.concat([SOTA_factor, new_factors], axis=1).dropna()
        else:
            combined_factors = new_factors

        # Sort and nest the combined factors under 'feature'
        combined_factors = combined_factors.sort_index()
        combined_factors = combined_factors.loc[:, ~combined_factors.columns.duplicated(keep="last")]
        new_columns = pd.MultiIndex.from_product([["feature"], combined_factors.columns])
        combined_factors.columns = new_columns
        # Due to the rdagent and qlib docker image in the numpy version of the difference,
        # the `combined_factors_df.pkl` file could not be loaded correctly in qlib dokcer,
        # so we changed the file type of `combined_factors_df` from pkl to parquet.
        target_path = exp.experiment_workspace.workspace_path / "combined_factors_df.parquet"

        # Save the combined factors to the workspace
        combined_factors.to_parquet(target_path, engine="pyarrow")

    result = exp.experiment_workspace.execute(
        qlib_config_name=f"conf.yaml" if len(exp.based_experiments) == 0 else "conf_combined.yaml"
    )

    exp.result = result

    return exp
```

Key features of the factor runner:
- Caches results to avoid redundant computation
- Processes factor data from experiment implementations
- Combines new factors with state-of-the-art (SOTA) factors
- Deduplicates factors based on information coefficients
- Executes the combined factors through Qlib's backtesting engine

The runner uses multiprocessing to handle multiple factor implementations efficiently and includes sophisticated error handling for empty or invalid factor data.

### Model Development

#### QlibModelRunner

The `QlibModelRunner` class in `model_runner.py` handles the execution of model experiments:

```python
def develop(self, exp: QlibModelExperiment) -> QlibModelExperiment:
    if exp.sub_workspace_list[0].file_dict.get("model.py") is None:
        raise ModelEmptyError("model.py is empty")
    # to replace & inject code
    exp.experiment_workspace.inject_files(**{"model.py": exp.sub_workspace_list[0].file_dict["model.py"]})

    env_to_use = {"PYTHONPATH": "./"}

    if exp.sub_tasks[0].model_type == "TimeSeries":
        env_to_use.update({"dataset_cls": "TSDatasetH", "step_len": 20, "num_timesteps": 20})
    elif exp.sub_tasks[0].model_type == "Tabular":
        env_to_use.update({"dataset_cls": "DatasetH"})

    result = exp.experiment_workspace.execute(qlib_config_name="conf.yaml", run_env=env_to_use)

    exp.result = result

    return exp
```

Key features of the model runner:
- Injects the model implementation code into the experiment workspace
- Configures environment variables based on the model type (TimeSeries or Tabular)
- Executes the model through Qlib's training and evaluation pipeline

### Feedback Generation

The feedback components in `feedback.py` evaluate experiment results and generate structured feedback:

#### QlibFactorExperiment2Feedback

Generates feedback for factor experiments:

```python
def generate_feedback(self, exp: Experiment, trace: Trace) -> HypothesisFeedback:
    # ...existing code...
    
    # Process the results to filter important metrics
    combined_result = process_results(current_result, sota_result)

    # Generate the system prompt
    sys_prompt = (
        Environment(undefined=StrictUndefined)
        .from_string(feedback_prompts["factor_feedback_generation"]["system"])
        .render(scenario=self.scen.get_scenario_all_desc())
    )
    # ...existing code...
```

The factor feedback generator:
- Processes results to extract important metrics
- Compares current results with SOTA results
- Uses templates from prompts.yaml to generate prompts
- Calls the LLM to analyze results and generate feedback
- Structures the feedback into observations, evaluations, and recommendations

#### QlibModelExperiment2Feedback

Generates feedback for model experiments:

```python
def generate_feedback(self, exp: Experiment, trace: Trace) -> HypothesisFeedback:
    # ...existing code...
    
    user_prompt = (
        Environment(undefined=StrictUndefined)
        .from_string(feedback_prompts["model_feedback_generation"]["user"])
        .render(
            context=context,
            last_hypothesis=SOTA_hypothesis,
            last_task=SOTA_experiment.sub_tasks[0].get_task_information() if SOTA_hypothesis else None,
            last_code=SOTA_experiment.sub_workspace_list[0].file_dict.get("model.py") if SOTA_hypothesis else None,
            last_result=SOTA_experiment.result if SOTA_hypothesis else None,
            hypothesis=hypothesis,
            exp=exp,
        )
    )
    # ...existing code...
```

The model feedback generator:
- Retrieves the SOTA hypothesis and experiment for comparison
- Provides the model implementation code for analysis
- Compares current results with previous results
- Generates structured feedback on the hypothesis performance

## Putting It All Together

The Qlib integration in RD-Agent demonstrates a complete research and development lifecycle for quantitative finance:

1. **Hypothesis Generation**: Using the proposal system to generate hypotheses
2. **Experiment Creation**: Converting hypotheses into executable experiments
3. **Factor/Model Loading**: Loading factor definitions from various sources
4. **Implementation**: Running the experiments through Qlib
5. **Feedback**: Analyzing results and generating feedback
6. **Iteration**: Using feedback to inform the next round of hypothesis generation

This integrated approach enables continuous improvement of financial factors and models through an automated, LLM-driven research process.