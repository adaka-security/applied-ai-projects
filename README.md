# AI Triage Agent

An LLM agent that classifies and prioritizes incoming items using
retrieval-augmented generation (RAG) and structured tool calling — turning
unstructured input into ranked, actionable decisions with clear reasoning.

This implementation applies the pattern to vulnerability (CVE) triage, but
the architecture generalizes to any "incoming item → prioritized decision"
workflow: support ticket routing, content moderation queues, alert
triage, lead scoring, and similar classification-and-prioritization tasks.

## Why this exists

Teams across security, support, and operations are flooded with incoming
items that need to be assessed, prioritized, and acted on — but doing this
manually is slow and inconsistent. This project demonstrates how an LLM
agent, grounded with retrieval over historical data, can automate that
first-pass triage: producing structured priority levels, reasoning, impact
assessment, and concrete next steps.

## Components

| File | Purpose |
|---|---|
| `src/ingest_cves.py` | Fetches recent CVEs from the NVD API and saves normalized JSON (swap for any ingestion source) |
| `src/vector_store.py` | Builds a ChromaDB vector store of item descriptions for RAG retrieval |
| `src/triage_agent.py` | Core agent: uses Claude with tool use to produce structured triage assessments, grounded with RAG context |
| `src/api.py` | FastAPI service exposing `/triage` and `/rebuild-index` endpoints |
| `data/cves_raw.json` | Sample dataset (10 real CVEs from 2023-2024) used to demonstrate the pipeline |

## Key techniques demonstrated

- **Tool use / structured output**: the agent uses Claude's tool-calling to return strictly-typed assessments (priority enum, reasoning, recommended actions) rather than free-text — making the output directly consumable by downstream systems.
- **RAG**: before assessing a new item, the agent retrieves the most similar historical items from a vector store, giving it pattern-matching context.
- **Context-aware reasoning**: the agent cross-references new items against provided contextual data (asset inventory) to ground its assessment in real-world specifics.
- **Production framing**: exposed via FastAPI so it can be integrated into existing tooling — triggered by a webhook, queue consumer, or scheduled job.

## Example domain: vulnerability triage

The included demo applies this pattern to CVE (vulnerability) triage, drawing on real-world security operations experience:

- **Input**: a CVE with description, CVSS score, and affected products
- **RAG context**: similar historical CVEs and their exploitation patterns
- **Output**: priority level (P0–P3), exploitability reasoning, business impact, and ordered remediation steps

## Setup

```bash
pip install -r requirements.txt
export ANTHROPIC_API_KEY=your_key_here
```

## Usage

### 1. Ingest data (optional — sample data included)

```bash
python src/ingest_cves.py
```

### 2. Build the vector store

```bash
python src/vector_store.py
```

### 3. Run triage

```bash
python src/triage_agent.py
```

Example output (real run on CVE-2024-21413, Microsoft Outlook "MonikerLink"):
[P0_IMMEDIATE] CVE-2024-21413

Impact: Remote code execution — potential ransomware, lateral movement,

credential harvesting via NTLM relay.

Reasoning: CVSS 9.8, network-based, no authentication required.

Matches known pattern of high-impact Outlook RCE bugs. Bypasses

Office Protected View. Active exploitation likely.

Remediation:

- Apply Microsoft February 2024 patch (KB5002otye)

- Push via WSUS/SCCM/Intune

- Disable Outlook Preview Pane as interim workaround

- Hunt for indicators in email gateway and EDR logs

- Block outbound SMB/WebDAV at perimeter firewall
See `triage_output.txt` for full output across all CVEs.

### 4. Run as an API

```bash
uvicorn src.api:app --reload
```

Visit `http://localhost:8000/docs` for interactive Swagger docs.

## Adapting to other domains

1. Replace `ingest_cves.py` with your own data source (tickets, alerts, leads)
2. Adjust the `TRIAGE_TOOL` schema in `triage_agent.py` for your priority levels
3. Update the prompt in `build_triage_prompt()` for your domain's reasoning criteria
4. Everything else (vector store, RAG, API) works unchanged

## Possible extensions

- Slack/email alerts for P0 findings
- Eval harness to measure triage consistency across model runs
- Multi-agent setup with a live CMDB instead of static inventory

## Tech stack

Python, Anthropic Claude API (tool use), ChromaDB, FastAPI, scikit-learn
