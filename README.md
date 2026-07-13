# saga-evaluation

# Evaluating Retrieval Quality Across LLMs in a Legal RAG Pipeline

This repository contains the evaluation framework developed for comparing retrieval quality across large language models in the Saga Legal Research pipeline.

The framework uses Langfuse traces to extract retrieved European Case-Law Identifiers (ECLIs) and compare them against verified reference ECLIs. It evaluates whether changing the generation model affects the retrieval of relevant Dutch case law.

The evaluated models are:

* GPT-5.5
* Claude Sonnet 4.6
* Gemini 3.1 Pro

## Overview

Saga's Legal Research pipeline processes a legal question in three main stages:

1. The selected language model reformulates the original question into multiple search queries.
2. The retrieval system searches the enabled legal data sources and returns candidate documents.
3. The model generates an answer using the retrieved material.

Each pipeline execution is recorded as a Langfuse trace. The notebooks in this repository connect these traces to a dataset of Dutch civil-law questions and evaluate the retrieved documents using their ECLI identifiers.

The evaluation is external to the Saga pipeline and does not modify its retrieval configuration.

## Repository Structure

```text
saga-evaluation/
├── evaluation.ipynb
├── script_evaluators.ipynb
├── saga-app.mov
└── README.md
```

### `evaluation.ipynb`

Connects to Langfuse and inspects the available datasets, traces, observations, and dataset runs.

The notebook is used to:

* authenticate the Langfuse client;
* retrieve and inspect pipeline traces;
* examine the structure of observations;
* connect traces to their corresponding dataset items;
* inspect retrieved documents and ECLI metadata;
* investigate whether generation inputs and outputs were logged correctly.

### `script_evaluators.ipynb`

Contains the deterministic ECLI-based evaluation workflow.

The notebook is used to:

* retrieve dataset items and associated traces;
* extract expected ECLIs from the dataset;
* extract retrieved ECLIs from trace observations;
* normalize ECLI representations;
* compare retrieved and expected ECLI sets;
* calculate retrieval precision, recall, and hit rate;
* store item-level scores in Langfuse;
* aggregate results for each model run.

### `saga-app.mov`

A short demonstration of the Saga Legal Research workflow used to generate the evaluated traces.

## Evaluation Dataset

The evaluation dataset contains 24 Dutch civil-law questions. Each item consists of:

* an input legal question;
* one or more expected ECLIs;
* a verified reference answer.

Example:

```json
{
  "input": {
    "query": "In a joint venture company with conflicting shareholder groups, may directors simply follow the interest of the shareholder by whom they were appointed or nominated?"
  },
  "expectedOutput": {
    "Sources": [
      "ECLI:NL:HR:2014:797"
    ],
    "correctAnswer": "No. Directors must be guided by the interest of the company and the enterprise connected with it."
  }
}
```

The reference ECLIs are treated as the ground-truth set for retrieval evaluation.

## Evaluation Metrics

For each dataset item, let:

* (R) be the set of ECLIs returned by the retriever;
* (G) be the expected ground-truth ECLI set.

The following metrics are calculated.

### Precision

The proportion of retrieved ECLIs that occur in the ground-truth set:

```text
precision = |R ∩ G| / |R|
```

### Recall

The proportion of expected ECLIs successfully retrieved:

```text
recall = |R ∩ G| / |G|
```

### Hit Rate

A binary score indicating whether at least one expected ECLI was retrieved:

```text
hit_rate = 1 if |R ∩ G| > 0 else 0
```

The metrics are calculated deterministically. No LLM-as-a-Judge model is used for ECLI matching.

## Experimental Runs

The same evaluation dataset was used for three model conditions:

| Run                | Model             | Evaluable items |
| ------------------ | ----------------- | --------------: |
| `saga-ecli-gpt`    | GPT-5.5           |              24 |
| `saga-ecli-claude` | Claude Sonnet 4.6 |              24 |
| `saga-ecli-gemini` | Gemini 3.1 Pro    |              23 |

The embedding model, retrieval index, and broader retrieval infrastructure remained unchanged. The selected generation model participated in query formulation and answer generation.

## Results

| Metric    | GPT-5.5 | Claude 4.6 | Gemini 3.1 |
| --------- | ------: | ---------: | ---------: |
| Precision |  0.0073 |     0.0108 |     0.0148 |
| Recall    |  0.1042 |     0.1597 |     0.1391 |
| Hit rate  |  0.1250 |     0.2083 |     0.2174 |
| (N)       |      24 |         24 |         23 |

All three model conditions achieved low ECLI retrieval scores. No model consistently outperformed the others across all metrics, suggesting that retrieval quality was a more important limitation than generation-model selection in the evaluated configuration.

## Requirements

* Python 3.10 or later
* Jupyter Notebook or JupyterLab
* Access to the relevant Langfuse project
* Langfuse API credentials

Install the principal dependencies with:

```bash
pip install jupyter langfuse pandas
```

Depending on the local environment and notebook version, additional packages imported by the notebooks may also need to be installed.

## Configuration

Set the Langfuse credentials as environment variables before running the notebooks.

### macOS or Linux

```bash
export LANGFUSE_PUBLIC_KEY="your-public-key"
export LANGFUSE_SECRET_KEY="your-secret-key"
export LANGFUSE_HOST="https://cloud.langfuse.com"
```

### Windows PowerShell

```powershell
$env:LANGFUSE_PUBLIC_KEY="your-public-key"
$env:LANGFUSE_SECRET_KEY="your-secret-key"
$env:LANGFUSE_HOST="https://cloud.langfuse.com"
```

An optional `.env` file may be used locally:

```env
LANGFUSE_PUBLIC_KEY=your-public-key
LANGFUSE_SECRET_KEY=your-secret-key
LANGFUSE_HOST=https://cloud.langfuse.com
```

Add it to `.gitignore`:

```gitignore
.env
.ipynb_checkpoints/
__pycache__/
```

## Usage

Clone the repository:

```bash
git clone https://github.com/suysal7/saga-evaluation.git
cd saga-evaluation
```

Start Jupyter:

```bash
jupyter lab
```

Run the notebooks in the following order:

1. Open `evaluation.ipynb`.
2. Configure and verify the Langfuse connection.
3. Inspect the dataset, traces, and observation structure.
4. Open `script_evaluators.ipynb`.
5. Select or configure the relevant dataset and run names.
6. Execute the ECLI extraction and evaluation cells.
7. Inspect the resulting item-level and aggregate scores in Langfuse.

The notebooks depend on data stored in the associated Langfuse project. They cannot reproduce the original experiment without access to the relevant datasets and traces.

## Methodology

For every model condition:

1. A Dutch civil-law question was submitted to Saga.
2. The selected model reformulated the question into one or more searches.
3. Saga retrieved candidate documents from the enabled legal sources.
4. The execution trace was recorded in Langfuse.
5. The trace was linked to the corresponding dataset item.
6. Retrieved ECLIs were extracted from the logged document metadata.
7. Retrieved ECLIs were compared with the expected ECLI set.
8. Precision, recall, and hit rate were stored and aggregated per run.

This design allows retrieval performance to be measured independently of subjective answer-quality judgements.

## Known Limitation

The original framework was intended to contain two evaluators:

* a retrieval evaluator, assessing whether the expected case law was retrieved;
* a generation evaluator, assessing whether the final answer cited the case law correctly.

However, the relevant generation observations contained no recorded input or output. Citation-level evaluation could therefore not be performed.

The reported metrics only show whether an expected document appeared among the retrieved candidates. They do not show whether that document was used correctly in the generated answer.

Additional limitations include:

* only 23–24 evaluable items per model run;
* strict exact matching of ECLI identifiers;
* relevant but unlisted cases receiving a score of zero;
* query-generation and retrieval effects not being independently isolated;
* evaluation on a limited set of Dutch civil-law exam questions.

## Future Work

Future development should focus on:

* instrumenting generation observations to record inputs and outputs;
* evaluating citation correctness in generated answers;
* expanding the number and legal coverage of dataset items;
* including real user queries;
* holding generated search queries constant across models;
* testing retrieval settings independently of the generation model;
* adding ranking-based metrics such as mean reciprocal rank and NDCG;
* supporting graded relevance judgements rather than exact binary matching.

## Research Context

This repository accompanies the thesis:

> *Evaluating Retrieval Quality Across LLMs in a RAG Pipeline for Legal Queries*

The work evaluates a Dutch legal RAG pipeline using Langfuse-based trace inspection and deterministic ECLI matching.

## Author

Sarp Uysal

## Disclaimer

The dataset and original Langfuse traces may contain private or internally provided material and are therefore not included in this public repository. The repository contains the evaluation implementation, but reproduction of the original runs requires authorized access to the corresponding Saga and Langfuse environments.
```
