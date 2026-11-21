# Promptly

A Python library that improves your LLM prompts automatically. Give it a rough prompt, and optionally a small labeled dataset. It rewrites the prompt, tests each version, and keeps the one that performs best on your task. It works with any LLM supported by LangChain.

## What you can use it for

- Improving prompts for classification or text-generation tasks without hand-tuning
- Generating labeled data for fine-tuning
- Building a repeatable way to score LLM response quality
- Tuning prompts used inside agent pipelines

## Features

- **Several optimization methods.** Choose between fast single-shot rewriting and slower evolutionary search, depending on your budget.
- **Works with any LLM.** Use OpenAI, open-source models, or any [LangChain-compatible model](https://python.langchain.com/docs/integrations/llms/).
- **No dataset? No problem.** Promptly can generate synthetic evaluation data when you don't provide any.
- **Built-in evaluation.** Score prompts with metrics for both classification and generation tasks.
- **Cost tracking.** See how many tokens and calls each optimization run used, and what it cost.
- **Automatic task detection.** Promptly infers the task type when you don't specify it.
- **Automatic method selection.** Let Promptly pick the best method for your task.
- **Extensible.** Add your own optimization method using the same interfaces.

## Optimization methods

Each method trades off differently between data needs, speed, quality, and cost. The ratings below are rough guides and depend on the task.

| Method                  | Needs data |  Runtime  | Quality gain |       Cost        |
| ----------------------- | :--------: | :-------: | :----------: | :---------------: |
| `hyper_light`           |     No     |    Low    |    Medium    |        Low        |
| `hyper`                 |    Yes     |  Medium   |     High     |      Medium       |
| `regps`                 |    Yes     |   High    |  Very high   |       High        |
| `rider`                 |    Yes     | Very high |  Very high   |     Very high     |
| `brave`                 |    Yes     |   High    |  Very high   | Budget-controlled |
| `sapo`                  |    Yes     |   High    |  Very high   |       High        |
| `compress`              |     No     |    Low    |    Medium    |        Low        |
| `reflective` _(legacy)_ |    Yes     |   High    |     High     |       High        |
| `distill` _(legacy)_    |    Yes     |   High    |     High     |       High        |

**Which one to pick:**

- Just want a better prompt quickly, with no data? Use `hyper_light` (the default).
- Have a small labeled dataset? Start with `hyper`.
- Want the best quality and can afford more API calls? Use `regps`, `sapo`, or `rider`.
- Need to stay within a fixed token budget? Use `brave`.
- Want a shorter prompt? Use `compress`.

## Installation

Requires **Python 3.12+**.

```bash
git clone https://github.com/zaynnmotiwala/promptly.git
cd promptly
pip install -e .
```

For development (tests and linting):

```bash
pip install -r requirements-dev.txt
```

## Quick start

By default, Promptly uses `gpt-4o-mini` through the OpenAI API. Set your key first:

```bash
# macOS / Linux
export OPENAI_API_KEY="your-key"
```

```powershell
# Windows (PowerShell)
$env:OPENAI_API_KEY = "your-key"
```

### Improve a prompt (no data needed)

```python
from coolprompt.assistant import PromptTuner

tuner = PromptTuner()
tuner.run("Write an essay about autumn")

print(tuner.final_prompt)
# You are an expert writer and seasonal observer tasked with composing a rich,
# well-structured, and vividly descriptive essay on the theme of autumn...
```

### Optimize against your own data

```python
final_prompt = tuner.run(
    "Classify the sentiment of the text: {text}",
    task="classification",
    dataset=["Great product", "Very disappointing"],
    target=["positive", "negative"],
    method="brave",
    problem_description="Classify product-review sentiment.",
    max_steps=20,
    initial_budget_tokens=50_000,
)
```

### Let Promptly choose the method

With `method="auto"`, Promptly profiles your task, finds similar benchmark datasets in its built-in metadata, and picks the method with the best balance of quality, cost, and runtime. The choice is saved in `tuner.meta_selection`.

```python
final_prompt = tuner.run(
    "Classify the sentiment of the text: {text}",
    task="classification",
    dataset=["Great product", "Very disappointing"],
    target=["positive", "negative"],
    method="auto",
    problem_description="Classify product-review sentiment.",
)
```

- To use your own metadata, pass `meta_classifier_path="/path/to/metadata.csv"`.
- Without `dataset` and `target`, `auto` falls back to `hyper_light`.

### Refine a generation prompt

```python
final_prompt = tuner.run(
    "Summarize the article concisely.",
    task="generation",
    dataset=["Article one", "Article two"],
    target=["Summary one", "Summary two"],
    method="sapo",
    validation_size=0.5,
    n_iterations=3,
    n_candidates=4,
)
```

### Use a different model

```python
from langchain_openai import ChatOpenAI
from coolprompt.assistant import PromptTuner

llm = ChatOpenAI(model="gpt-4o", temperature=0)
tuner = PromptTuner(target_model=llm)
```

## Examples

The [`notebooks/examples`](notebooks/examples/) folder has ready-to-run notebooks:

| Notebook                  | Shows                                          |
| ------------------------- | ---------------------------------------------- |
| `basic.ipynb`             | Minimal end-to-end usage                       |
| `methods.ipynb`           | Running and comparing the optimization methods |
| `metrics.ipynb`           | Built-in evaluation metrics                    |
| `method_evaluation.ipynb` | Benchmarking methods on a dataset              |
| `cost_track.ipynb`        | Tracking tokens, calls, and cost               |
| `custom_apo_method.ipynb` | Writing your own optimization method           |

## Project structure

```text
├── coolprompt/
│   ├── assistant.py          # PromptTuner: the main entry point
│   ├── optimizer/            # Optimization methods (hyper, regps, rider, brave, sapo, ...)
│   ├── evaluator/            # Metrics and scoring
│   ├── method_evaluation/    # Benchmarking methods on datasets
│   ├── meta_selector/        # Automatic method selection (method="auto")
│   ├── spec_generator/       # Synthetic data and task specs
│   ├── task_detector/        # Automatic task detection
│   ├── language_model/       # Default LLM setup
│   └── utils/                # Shared helpers
├── notebooks/examples/       # Usage examples
├── docs/                     # API reference and contributing guide
└── test/                     # Unit tests
```
