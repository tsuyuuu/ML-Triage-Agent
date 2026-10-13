# ML-Triage-Agent
AI Security Triage Agent built with Ollama Cloud for CYBERSEC590: 
ML in Modern Cyber Operations at Duke University. 

**Team:** Parker Cummins, Lalit Lakamsani, Jenessa Lu, Vamsee Vusuwandla

## Member Contributions
Parker: Created workflow diagram, rag search function <br />
Vamsee: Assisted with document creation, defining evaluation tables, wrote generate_report function <br />
Jenessa: Wrote cisa_kev function, agent evaluation, tool evaluation. <br />
Lalit: Assisted with document creation, wrote generate_report function. <br />

Everyone: Wrote system prompt, came up with ideas for how to go about the project/writing overall code

## Overview

This project uses **Ollama Cloud (`gpt-oss:20b`)**, LangChain tool calling, and Python to investigate one security question in a **simulated** Windows enterprise: **What is the risk of SMB signing not being required on FILE04 at Northbridge Services?**

The agent retrieves environment evidence and Microsoft SMB guidance, evaluates prerequisites for a potential SMB relay attack, and saves a JSON assessment for human review. This is an **evidence-based triage exercise**, not a vulnerability scan or proof of exploitation.

## Assessment scope

The supplied Northbridge environment documents the following:

- **FILE04** belongs to the **W05 file-server group** (Windows Server 2022, SMB 3).
- SMB signing is **not required** on FILE04; SMB1 is disabled across W05 (**W05, S03**). *Not required* does not mean every SMB session is unsigned.
- No SMB service is exposed to the public internet, and inter-zone traffic is denied by default unless explicitly allowed (**S02**).
- The evidence does **not** establish internal attacker reachability to FILE04, authentication-relay prerequisites, whether an unsigned connection can be negotiated, or the classification and permissions of FILE04's shares.

The scenario describes departmental shares at the W05 **group** level. It does not establish that Finance or HR data specifically resides on FILE04.

## How it works

1. Load and extract text from the Northbridge Word document.
2. Retrieve current Microsoft SMB signing and security-hardening documentation.
3. Use TF-IDF and cosine similarity to retrieve relevant passages from the environment or guidance collection.
4. Let the LLM request registered tools; Python validates, executes, and returns their results to the model.
5. Evaluate SMB relay prerequisites without treating unknown conditions as confirmed.
6. Generate `risk_assessment.json` and preserve the run's tool calls and outputs.

The agent permits up to **10 model iterations**. Tool selection is model-driven, so it is not guaranteed that every possible tool will be called. A run that stops without a successful `generate_report` result is incomplete.

### Tools

| Tool | Purpose | Main inputs | Result |
| --- | --- | --- | --- |
| `rag_search` | Search environment passages or Microsoft guidance using TF-IDF | `query`, `collection`, `top_k` | Ranked passages, source identifiers/URLs, and retrieval metadata where available |
| `lookup_cisa_kev` | Look up a specified CVE in CISA's Known Exploited Vulnerabilities catalog | `cve_id` | `found`, `not_listed`, `unavailable`, or validation error |
| `check_attack_applicability` | Evaluate the prerequisites for the FILE04 SMB relay scenario | SMB signing requirement, attacker reachability, relay prerequisites, unsigned session feasibility | `applicable`, `not_applicable`, or `insufficient_evidence`, with conditions and unknowns |
| `generate_report` | Apply risk-rating logic and save a structured assessment | Asset, threat, applicability, impact, rationale, evidence, unknowns, actions | `risk_assessment.json` |

**About CISA KEV:** The core FILE04 case is a configuration weakness rather than a claim about a particular CVE, so the successful baseline run does **not** call the KEV tool. The notebook separately tests this tool and includes a CVE-focused investigation using **CVE-2025-33073**. A KEV listing indicates known exploitation of a CVE, **not** that FILE04 is affected; the existing applicability checker evaluates the SMB relay scenario, not CVE-specific patch or version applicability.

## Risk-rating logic

The applicability checker uses `True` (supported), `False` (necessary condition absent), and `None` (unknown):

| Applicability | Meaning |
| --- | --- |
| `applicable` | All assessed prerequisites are confirmed |
| `not_applicable` | At least one necessary prerequisite is confirmed absent |
| `insufficient_evidence` | A decisive prerequisite is unknown, without a confirmed disqualifier |

For an **applicable** attack path with supporting evidence, the reporter assigns a risk rating based on potential business impact:

| Impact category | Risk rating |
| --- | --- |
| `critical` — critical systems or high-consequence financial/administrative effects | High |
| `departmental` — bounded departmental effects | Medium |
| `minor` — limited consequences | Low |

If evidence is missing or applicability is `insufficient_evidence`, the reporter returns **Insufficient Evidence**. If applicability is `not_applicable`, it returns **Not Applicable**, **not Low**. The final assessment must be reviewed against the group's documented policy and supporting evidence.

## Demonstrated FILE04 result

The preserved successful run executed **four tool calls**: environment `rag_search`, guidance `rag_search`, `check_attack_applicability`, and `generate_report`.

| Field | Result |
| --- | --- |
| Asset | `FILE04` |
| Threat | Potential SMB relay attack associated with SMB signing not being required |
| Applicability | `insufficient_evidence` |
| Final risk rating | Insufficient Evidence |
| Business impact estimate | `departmental` (provisional) |
| Analyst review | `pending` |

**Why:** The signing requirement exception is documented, but an exploitable SMB relay path has **not** been demonstrated. Internal attacker reachability, relay prerequisites, and unsigned SMB negotiation remain unknown. FILE04's share permissions and data classification are also unverified.

**Next steps:** Review internal access paths and firewall policy; verify effective SMB signing configuration and negotiated sessions through authorized methods; investigate authentication-relay prerequisites; and identify the data and permissions on FILE04. These are **recommendations**, not tests or changes performed by the agent.

The report preserves environment evidence references (**W05, S02, S03**) and uncertainty. The run history also preserves the retrieved Microsoft source URLs and timestamps, even when those external sources are not included in the final report's `evidence` list.

## Running in Google Colab

The notebook was developed for **Google Colab**. Its `google.colab` imports and `!pip` commands mean an exported `.py` file is **not directly runnable as a standard Python script** without adaptation.

1. Open `ML_Triage_Agent.ipynb` in Colab.
2. Install the project dependencies from `requirements.txt`:

   ```python
   !pip install -r requirements.txt
   ```

3. Create a Colab **Secret** named `OLLAMA`, paste your Ollama Cloud API key, and grant the notebook access. Do **not** include the key in GitHub or saved outputs.
4. Run the setup cells. The notebook configures `ChatOllama` with `gpt-oss:20b` and `temperature=0`.
5. When prompted, upload `Windows_Enterprise_Environment.docx`.
6. Run the retrieval, tool-definition, agent, and evaluation cells in order.
7. Inspect the generated assessment, tool history, and test output. Preserve the final files for the submission.

The notebook currently retrieves Microsoft guidance **live** using HTTP requests; it does not automatically load a cached snapshot if the requests fail. It also requires an available Ollama Cloud connection. The `requirements.txt` should contain the package versions tested in your Colab runtime; Python's built-in modules and `google.colab` do not need to be listed as pip dependencies.

## Tests and Evaluation

The agent was evaluated using controlled tests to verify attack applicability, risk-rating consistency, insufficient-evidence handling, and API failure recovery. **All implemented tests passed**, with actual results matching expected outcomes.

| **Test** | **Expected Result** | **Actual Result** | **Status** |
| --- | --- | --- | --- |
| All SMB relay prerequisites confirmed | `applicable` | `applicable` | PASS |
| Necessary network access blocked | `not_applicable` | `not_applicable` | PASS |
| Necessary reachability unknown | `insufficient_evidence` | `insufficient_evidence` | PASS |
| Confirmed applicable path with critical impact | `High` | `High` | PASS |
| Confirmed applicable path with departmental impact | `Medium` | `Medium` | PASS |
| Confirmed applicable path with minor impact | `Low` | `Low` | PASS |
| FILE04 prerequisites unresolved | `insufficient_evidence` | `insufficient_evidence` | PASS |
| Simulated CISA KEV API connection failure | `unavailable` without an uncaught exception | `unavailable` without an uncaught exception | PASS |

### End-to-End Agent Evaluation

A complete agent execution was successfully demonstrated. The agent:

1. Retrieved Northbridge environment evidence through RAG.
2. Retrieved external Microsoft SMB security guidance.
3. Evaluated SMB relay attack prerequisites using the applicability checker.
4. Generated a structured JSON risk assessment with supporting evidence, unknowns, and recommended actions.

The final assessment assigned FILE04 a risk rating of **Insufficient Evidence**, consistent with the group's policy for cases where decisive attack prerequisites remain unverified. Departmental business impact was identified as provisional.

The generated report records `analyst_review: pending` by default. Human verification is a separate step from automated testing and does not indicate a test failure.

### Reproducibility

The saved agent baseline includes:

- Model: `gpt-oss:20b`
- Provider: Ollama Cloud
- Temperature: `0`
- Environment filename
- System prompt
- Execution timestamp
- Tool-call arguments and results

The notebook, original Northbridge environment document, and generated JSON outputs are retained separately to support reproducibility and analyst review.

## Outputs

The notebook can produce:

- `risk_assessment.json` — risk decision, rationale, provisional impact, evidence, unknowns, evidence needed, recommended action, and analyst review state.
- `tool_history.json` — executed tool names, arguments, and results.
- `agent_baseline.json` — run metadata and the tool execution history.
- `microsoft_smb_guidance.json` — retrieved guidance passages, URLs, timestamps, and source modes.

Files are written to the Colab working directory and must be downloaded or copied into the submission repository deliberately. Keep generated outputs from the **same completed run** together; individual reporter tests can overwrite `risk_assessment.json` unless it is restored afterward.

## Limitations

- Northbridge is **simulated**. The agent performs no live enterprise scanning, authentication testing, exploitation, or remediation.
- TF-IDF retrieval can surface tangential passages; citations and LLM-generated claims are **not automatically validated** against the original sources.
- The applicability tool is tailored to this SMB relay scenario, not a general CVE applicability engine.
- Missing prerequisites produce an **Insufficient Evidence** decision even though the configuration exception itself is confirmed.
- Departmental impact is **provisional**, pending FILE04-specific data classification and share-permission evidence.
- External guidance scraping can include duplicated or page-navigation text; failed live retrieval is not currently replaced by an automatic cached fallback.
- The model may stop early without completing every tool call; completion must be verified from the saved history.
- An analyst must review the final rating and source evidence before relying on it.

## AI assistance disclosure

ChatGPT **GPT-6** assisted with Python function drafting, error handling, Ollama/LangChain tool integration, debugging, and documentation on **October 12, 2026**. The group defined the threat scope, intended tool interfaces, and applicability/impact rating criteria and is responsible for reviewing, testing, and validating the code, supporting sources, and final conclusions. Specific code-cell disclosures are included in the notebook where applicable.

## References

- Microsoft Learn, [Overview of SMB signing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview)
- Microsoft Learn, [SMB security hardening](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-security-hardening)
- CISA, [Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- Course-provided, *Windows Enterprise Environment* (Northbridge Services; simulated; September 23, 2026)
