# AIVANTA AI

Our own AI: a language model trained from zero on our 50 GB of books, RAG answers with book and page references, and agents. No outside models and no Ollama. Served to the frontend through the AI API.

The AI talks to the frontend and backend only through `serving/` (REST + SSE / WebSocket). The frontend never calls models or the database directly.

## Top level

The top level matches `frontend/`, so both teams find things in the same place.

```
ai/
├── .vscode/            Shared editor settings for the team
├── environments/       Environment files (development, staging, production) - .env.example only
├── docs/               Project documentation
│   ├── architecture/   How the AI system is put together, diagrams
│   ├── adr/            Architecture Decision Records, one file per decision
│   ├── api-contracts/  Agreed request / response contracts with backend + frontend
│   ├── standards/      Coding, review, data-handling and model-release standards
│   ├── guides/         How-to guides
│   ├── onboarding/     First-week material for new AI team members
│   ├── runbooks/       What to do when something breaks in production
│   ├── releases/       Release notes per model / service version
│   ├── model-cards/    One card per released model
│   ├── data-cards/     One card per dataset
│   └── experiments/    Write-ups of finished experiments
├── deployment/         docker, nginx, systemd, kubernetes, ci, oracle, monitoring
├── scripts/            setup, data, training, quality, release
├── templates/          Starting points: agent, tool, experiment, pipeline
├── tests/              unit, integration, db, api, evaluation, performance, security, fixtures, mocks
│
├── configs/            YAML settings: model, tokenizer, data, training, rag, agents, serving, evaluation
├── data/               NOT in Git - raw, interim, processed, annotations, manifests
├── models/             NOT in Git - tokenizer, checkpoints, embeddings, released, exported
├── notebooks/          exploration, experiments, reports
├── pipelines/          Jobs that chain the package from books to a released model
└── src/aivanta_ai/     Application source (below)
```

`data/` and `models/` are too big for Git. The folders are kept so everyone uses the same paths; the real files live on the Oracle block volume and Object Storage (see `.gitignore`).

## Data flow

```
data/raw/books ──► data/interim/extracted-text ──► data/interim/cleaned
                                                     │
               ┌─────────────────────────────────────┼──────────────────────────┐
               ▼                                     ▼                          ▼
 data/processed/pretraining-shards     data/processed/rag-chunks     data/processed/finetuning
               │                                     │                          │
               ▼                                     ▼                          │
 models/tokenizer ──► models/checkpoints     Oracle BOOK_CHUNK (vectors)        │
               │                                     │                          │
               └──────────────► fine-tune ◄──────────┼──────────────────────────┘
                                   │                 │
                                   ▼                 ▼
                         models/released ──► serving/ (chat = RAG + agents) ──► frontend
```

## Source

```
src/aivanta_ai/
├── core/            Infrastructure, no AI logic
│   ├── config/  logging/  observability/  storage/  security/  exceptions/  utils/
├── data_pipeline/   Books -> clean text
│   ├── ingestion/  extraction/  cleaning/  deduplication/  filtering/  sharding/  validation/
├── tokenizer/       Our own BPE tokenizer: training/, encoding/
├── llm/             Our own model
│   ├── architecture/  The transformer
│   ├── pretraining/   Learn language from the books
│   ├── finetuning/    Learn to answer questions (SFT)
│   ├── alignment/     Prefer good answers (DPO)
│   ├── inference/     Generate text
│   └── export/        GGUF / quantized for llama.cpp
├── embeddings/      Text -> vectors: training/, encoding/
├── rag/             Retrieval-augmented answers
│   ├── chunking/  indexing/  vector_store/  retrieval/  reranking/  prompting/  citation/  pipeline/
├── agents/          Agents and their tools
│   ├── base/  orchestration/  catalog/  tools/  memory/  registry/  policies/
├── safety/          input_guards/, output_guards/, red_team/
├── evaluation/      metrics/, benchmarks/, reports/
├── serving/         The AI API
│   ├── api/routes/  api/schemas/  api/middleware/  streaming/  workers/
├── mlops/           experiment_tracking/, model_registry/, monitoring/, feedback/
└── cli/             aivanta-ai ingest | train | index | evaluate | serve
```

### Pipelines

| Pipeline | In | Out |
| --- | --- | --- |
| `ingest_books` | `data/raw/books` | `data/interim/cleaned` |
| `build_tokenizer` | cleaned text | `models/tokenizer` |
| `pretrain` | pre-training shards | base model in `models/checkpoints` |
| `finetune` | base model + Q&A sets | chat model |
| `build_index` | cleaned text | chunks + vectors in Oracle `BOOK_CHUNK` |
| `evaluate` | model + index | scores, pass / fail against `configs/evaluation` |
| `release` | approved model | `models/released/vX.Y.Z` + deploy |

### How the frontend lines up

| Frontend feature (`frontend/src/modules/...`) | AI code it calls |
| --- | --- |
| `agent-studio/agents`, `agent-versions` | `agents/catalog`, `agents/registry` |
| `agent-studio/tools` | `agents/tools` |
| `agent-studio/knowledge-base` | `rag/indexing`, `data_pipeline` |
| `agent-studio/memory-context` | `agents/memory` |
| `agent-studio/models` | `mlops/model_registry` |
| `agent-studio/ai-safety` | `safety/` |
| `simulation-console/execution-monitor` | `serving/streaming` |
| `evaluation-center/*` | `evaluation/`, `mlops/experiment_tracking` |
| `analytics/agent-metrics` | `mlops/monitoring` |

Every one of these goes through `serving/api/routes`, and the request / response shapes are written down in `docs/api-contracts/` (the same file in both repos).

### Adding something new

- **New agent:** copy `templates/agent` to `src/aivanta_ai/agents/catalog/<name>`, set its allowed tools in `configs/agents/`, add tests in `tests/unit/agents/`.
- **New tool:** copy `templates/tool` to `src/aivanta_ai/agents/tools/<name>` and register it in `agents/registry`.
- **New experiment:** copy `templates/experiment`; when finished, write it up in `docs/experiments/`.
- **New pipeline:** copy `templates/pipeline` to `pipelines/<name>`.

## Rules

1. **Imports go one way:** `serving` → `agents` → `rag` → `llm` / `embeddings` → `tokenizer` → `data_pipeline` → `core`. Nothing lower imports from anything higher.
2. **Everything outside goes through `core/`.** Database, Object Storage, files, secrets: only `core/storage` and `core/security` touch them.
3. **No numbers in code.** Sizes, paths, top-k, learning rates: all in `configs/`. Secrets in `environments/` (never in Git).
4. **Evaluation data is never trained on.** `data/processed/evaluation` is read only by `evaluation/` and `tests/evaluation/`.
5. **A model goes live only from `models/released/`**, after `pipelines/evaluate` passes and the release is approved.
6. **Unit tests mirror `src/`**: `src/aivanta_ai/rag/chunking` → `tests/unit/rag/chunking`.

## Naming

| Item | Convention | Example |
| --- | --- | --- |
| Folders outside `src/` | kebab-case | `model-cards` |
| Python packages / modules (and `pipelines/`) | snake_case | `vector_store`, `build_index`, `hybrid_search.py` |
| Classes | PascalCase | `BookResearchAgent` |
| Functions, variables | snake_case | `retrieve_chunks()` |
| Configs | kebab-case `.yaml` | `llm-small.yaml` |
| Tests | `test_` + name | `test_hybrid_search.py` |
| Released models | semantic version | `models/released/v1.2.0` |

Empty folders hold a `.gitkeep` file so Git tracks them (Python packages hold an empty `__init__.py`); delete `.gitkeep` once the folder has real files.
