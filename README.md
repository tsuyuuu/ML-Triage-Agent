# ML-Triage-Agent
AI Security Triage Agent built with Ollama Cloud for CYBERSEC590: 
ML in Modern Cyber Operations at Duke University. 

**Team:** Parker Cummins, Lalit Lakamsani, Jenessa Lu, Vamsee Vusuwandla

## Overview

This project implements a Python-based, Ollama-powered security triage 
agent that assesses the potential risk of an **SMB relay attack against 
FILE04** in the fictional **Northbridge Services** Windows enterprise 
environment. The agent retrieves internal environment evidence and external 
Microsoft SMB guidance, checks attack prerequisites, and produces a structured, 
evidence-supported risk assessment for analyst review.

The project assesses **one defined weakness**, not the security of the 
entire enterprise. All Northbridge records are **simulated teaching data**; 
no real enterprise scan or exploitation was performed.

## Assessment scope

**Risk Question:** What risk does SMB signing not being required on FILE04 
pose to Northbridge Services, considering potential SMB relay attacks, network 
access restrictions, and business impact?

- **Target:** FILE04 in the W05 file-server group (Windows Server 2022, SMB 3).
- **Confirmed configuration:** SMB signing is not *required* on FILE04; SMB1 is disabled across W05 (W05, S03). This does **not** prove every session is unsigned.
- **Network context:** No Internet-facing SMB; inter-zone network access is denied by default unless permitted (S02).
- **Unresolved:** Internal attacker reachability, authentication-relay prerequisites, unsigned SMB negotiation, FILE04's exact data classification and share permissions.

The environment is fictional and simulated. The project does not scan,
exploit, or modify a real network. Suggested verification activities in
the assessment are recommendations, not actions performed by the agent.

## Architecture

The implementation uses:

- **Ollama Cloud:** `gpt-oss:20b`.
- **Python and LangChain:** tool orchestration and message handling.
- **TF-IDF and cosine similarity:** retrieval from environment and
  Microsoft guidance passages.
- **Python policy functions:** applicability checks and rating assignment.
- **Manual analyst review:** verification of evidence and the final decision.

**Workflow:** 

1. Upload and parse the Northbridge Word document.
2. Retrieve Microsoft SMB signing and hardening documentation.
3. Let the model select tools and construct arguments.
4. Execute the requested tools in Python.
5. Return tool results to the model.
6. Evaluate applicability and generate the assessment.
7. Save the assessment and tool history for analyst review.

The agent permits up to 10 model iterations. An iteration may contain
multiple tool calls. Text-only completion or reaching the limit without
a report is an incomplete run.

See the design document and workflow PDF for additional detail.

## Tools

| Tool | Purpose | Inputs | Output |
| --- | --- | --- | --- |
| `rag_search` | Retrieve evidence from the Northbridge document or saved Microsoft guidance using TF-IDF ranking | `query`, `collection` (`environment`/`guidance`), `top_k` | Ranked passages, sources, retrieval metadata where available |
| `lookup_cisa_kev` | Check whether a **specific CVE** appears in CISA's Known Exploited Vulnerabilities catalog | `cve_id` | Catalog record, not-listed result, or explicit lookup failure |
| `check_attack_applicability` | Compare the SMB relay prerequisites supplied from retrieved evidence | Signing requirement, reachability, relay prerequisites, unsigned negotiation (`True`/`False`/`None`) | `applicable`, `not_applicable`, or `insufficient_evidence`, with unknowns |
| `generate_report` | Apply the group's policy and serialize the final assessment | Asset, threat, applicability, impact, rationale, evidence, unknowns, action | `risk_assessment.json` |

CISA KEV is conditional. The configuration-only SMB signing case does
not require a CVE. A KEV listing does not establish local applicability,
and absence from KEV does not establish safety.

## Risk-rating policy

Prerequisites are represented as `True`, `False`, or `None`:

- `True`: condition supported as present.
- `False`: necessary condition supported as absent.
- `None`: condition unknown.

The applicability checker returns:

- `applicable` when all assessed prerequisites are confirmed.
- `not_applicable` when a necessary prerequisite is confirmed absent.
- `insufficient_evidence` when unresolved prerequisites remain and none
  establishes non-applicability.

The group's policy assigns a rating when an attack path is supported by evidence and its potential business impact is 
identified:

| Applicability and impact | Decision |
| --- | --- |
| Supported path affecting critical services or sensitive financial/administrative data | **High** |
| Supported path affecting bounded departmental data/services | **Medium** |
| Supported path with only minor consequences | **Low** |
| Decisive applicability information is missing or conflicting | **Insufficient Evidence / Review** |

**Important limitation:** The risk assessment distinguishes a confirmed configuration weakness from a confirmed attack path. For FILE04, SMB signing is not required, but internal reachability, authentication relay prerequisites, and unsigned SMB negotiation remain unverified. Therefore, the agent assigns Insufficient Evidence rather than assuming exploitability. A not_applicable finding is reported separately as Not Applicable, rather than automatically receiving a Low risk rating. Business impact remains provisional until FILE04's data classification and share permissions are verified.

## Repository files

| File | Description |
|---|---|
| `ML_Triage_Agent(1).ipynb` | Colab implementation, demonstration, and test cells |
| `Windows_Enterprise_Environment.docx` | Required simulated environment input; include this supplied document |
| `Assignment 3_ Build a Security Triage Agent.docx` | Design and tool descriptions |
| `SMB Signing Triage Agent Workflow (FILE04) 1.pdf` | Workflow diagram |
| `risk_assessment(1).json` | Saved example assessment |
| `agent_baseline(1).json` | Saved prompt, model configuration, and tool execution history |
| `microsoft_smb_guidance.json` | Retrieved Microsoft guidance passages |

Fresh notebook runs generate `risk_assessment.json`,
`tool_history.json`, and `agent_baseline.json`.

## Setup and execution (Google Colab)

The current notebook is designed for **Google Colab**. It uses
`google.colab.files` for uploads and downloads and
`google.colab.userdata` for secrets.

Required packages include:
- `ollama`
- `langchain-ollama`
- `langchain-core`
- `requests`
- `beautifulsoup4`
- `python-docx`
- `scikit-learn`

1. Open the project notebook (`.ipynb`) in Google Colab. **Add the notebook to this repository before submission.**
2. Install dependencies used in the notebook:

   ```python
   !pip install -q langchain-ollama python-docx scikit-learn requests beautifulsoup4
   ```

3. In **Colab → Secrets**, create a secret named `OLLAMA` with your Ollama Cloud API key and grant notebook access. **Never commit or print the key.**
4. Configure `ChatOllama` using model `gpt-oss:20b`, the Ollama Cloud endpoint, and the key loaded from Secrets. The API key is authenticated through the notebook's configured authorization header.
5. Upload `Windows_Enterprise_Environment.docx` using `files.upload()` when prompted. Load the Microsoft guidance collection using the notebook's retrieval code (or the saved evidence snapshot, if the notebook is configured to read it).
6. Run the cells defining the four tools, bind them to the LLM, and run the `run_agent(...)` cell for the FILE04 SMB question.
7. Review `risk_assessment.json` and the recorded tool history. The generated assessment is **pending human review**, not an automated remediation instruction.

The notebook assumes access to Ollama Cloud and that the selected model supports tool calling. Internet access is needed
for live guidance/KEV lookups. Failed lookups should be reported as unavailable rather than interpreted as evidence of 
safety.

Do not commit API keys or other secrets.

### Dependency versions and configuration

Python and package versions depend on the Colab runtime. For reproducibility, record the actual environment used for the final run:

```python
import sys
print(sys.version)
!pip freeze > requirements-lock.txt
```

Include `requirements-lock.txt` with the submission. The model recorded in the saved baseline is `gpt-oss:20b`; the baseline does not independently establish the exact library versions or all model settings.

## Demonstrated result

The saved successful run reports:

- **Asset:** FILE04
- **Threat:** SMB relay attack
- **Applicability:** `insufficient_evidence`
- **Risk rating:** `Insufficient Evidence`
- **Impact used by the reporter:** `departmental` (**provisional** because the precise FILE04 shares and sensitivity are unknown)
- **Recommended follow-up:** Verify internal reachability, effective SMB signing behavior, relay prerequisites, and file classifications/share permissions; review appropriate SMB signing enforcement and firewall access rules.

The saved tool history shows **four executed calls**, in order: environment RAG, Microsoft guidance RAG, applicability checker, and report generator. The last call successfully returned a structured assessment. The current conclusion does **not** assert a confirmed exploitable relay attack.

## Testing and evaluation

Individual function tests were run in Colab and were reported as passing. They cover:

1. **Applicability changes** — confirmed vs blocked vs unknown prerequisites.
2. **Impact changes** — critical, departmental, and minor consequences under confirmed applicability.
3. **Missing evidence** — unknown decisive prerequisites result in insufficient evidence.
4. **Controlled failure** — a simulated unavailable CISA KEV API is handled without an uncaught crash.

**Before submission:** Add the actual **expected-versus-actual results and pass/fail evidence** from the notebook. The saved baseline documents the successful assessment, not all four standalone test outputs. Testing of the *entire agent's* behavior during an external-tool failure is a separate limitation if only the KEV function was unit-tested.

## Files and artifacts

| File | Description |
| --- | --- |
| Project notebook (`.ipynb`; add before submission) | Imports, model setup, tools, agent loop, tests |
| `Windows_Enterprise_Environment.docx` | Original simulated enterprise scenario |
| `microsoft_smb_guidance.json` | Saved external guidance passages, source URLs, and retrieval timestamps |
| `risk_assessment.json` | Final structured FILE04 assessment |
| `agent_baseline.json` | Model/provider details and ordered tool requests/results |
| `README.md` | Setup, architecture, results, limitations and disclosures |
| Design/architecture diagram (add before submission) | Workflow, decision points, tool contracts, and analyst review |
| Test results (add before submission) | Expected-versus-actual policy checks and controlled failure |

Use consistent filenames when copying the final downloaded files into the repository (the browser may add `(1)` or `(2)` to duplicate downloads). Keep the same successful report and baseline together.

## Limitations and analyst review

- This is a simulated exercise, **not** a live network scan or an exploitation finding.
- The RAG implementation uses TF-IDF; it may rank headings or tangential passages highly. It does not validate truth or completeness.
- LLM-supplied applicability inputs and report citations are not automatically verified against every original source. A human must check them.
- Internal FILE04 reachability, effective unsigned session negotiation, authentication-relay conditions, and exact share sensitivity remain unresolved.
- The Microsoft guidance snapshot includes some duplicate and page-navigation text; preserving an unchanged original plus a separately cleaned copy would improve future retrieval.
- The KEV catalog is relevant only to specific CVEs; absence from KEV does **not** mean a weakness is safe.
- The reporter's provisional departmental impact and the `not_applicable` → Low mapping should be treated as documented policy limitations.

## Team contributions

**Team members:** Parker Cummins, Lalit Lakamsani, Jenessa Lu, Vamsee Vusuwandla.

**Complete before submission:** Record each member's actual responsibilities (e.g., threat selection, architecture/rating policy, tool implementation, testing, evidence review, demo, documentation). Do not assign contributions that have not been confirmed by the team.

## AI assistance disclosure

ChatGPT **GPT-6** assisted with Python function drafting, error handling, Ollama/LangChain tool integration, debugging, and documentation on **October 12, 2026**. The group defined the threat scope, intended tool interfaces, and applicability/impact rating criteria and is responsible for reviewing, testing, and validating the code, supporting sources, and final conclusions. Specific code-cell disclosures are included in the notebook where applicable.

## References

- Microsoft Learn, [Overview of SMB signing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview)
- Microsoft Learn, [SMB security hardening](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-security-hardening)
- CISA, [Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- Course-provided, *Windows Enterprise Environment* (Northbridge Services; simulated; September 23, 2026)
