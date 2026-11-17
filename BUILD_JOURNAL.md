# Build Journal — October 2026

**NovaMart: Multi-Agent Customer Support System** · Future AWS Agent Engineer Nanodegree (Udacity + AWS) capstone · passed capstone review.

> A restructured, searchable version of [`NovaMart_Build_Journal.pdf`](NovaMart_Build_Journal.pdf). It covers what the course supplied, what I implemented, how the two build sessions went, and what I'd improve next time.

---

## 1. What was expected — and what I did

The course provided the working scaffold. My job was to complete the agent system, deploy it, and prove it worked.

| Provided by the course | Implemented by me |
|---|---|
| CloudFormation stack: DynamoDB tables, S3 buckets, S3 Vectors bucket + indexes, IAM roles | The five agents, their tools, system prompts and routing rules in `src/agent_orchestrator.py` |
| `seed_data.py`, `cleanup.py`, `config.py` | Three retriever sub-agents and the parallel `search_all_policies` tool |
| WorkflowState helpers with optimistic locking | Guardrail creation and numbered versioning |
| `agent_utils.py` (terminal trace UI), `agent_observability.py` (X-Ray `@tool` wrapper) | Runtime deployment function and environment variables for AgentCore Runtime |
| `agentcore_cli.py`, `bedrock_kb_retrieval.py`, `demo.py` | AgentCore Memory configuration |
| Automated test suite | CloudWatch and X-Ray configuration |
| — | Three Knowledge Bases created and synced in the console, `.env` wiring, and end-to-end testing |

**In one line:** the course gave me the infrastructure, the helper modules and the tests; I built the behaviour, deployed it, configured the AWS-side resources, and ran the scenarios.

## 2. The build sessions, in order

Reconstructed from screenshot timestamps in `screenshots/` and AgentCore CLI logs in `project/starter/agentcore/.cli/logs/`.

**Evening of 19 October — get it running in AWS**

| Approx. time | Activity | Evidence |
|---|---|---|
| 21:09 → 21:56 | Five deployment iterations through the AgentCore CLI (guardrail versioning, runtime, memory, observability) | `deploy-20261019-*.log` (5 files) |
| 21:58 | First invocation against the deployed runtime | `invoke-agentcore_runtime-20261019-215824.log` |

**Evening of 20 October — data, knowledge bases, scenarios, proof**

| Approx. time | Activity | Evidence |
|---|---|---|
| 19:51 | Seeded DynamoDB: 4 customers, 11 orders, 6 policy documents to S3 | [Figure 1](screenshots/seeding_data.png) |
| 19:55 → 20:07 | Created and synced the three Knowledge Bases in the Bedrock console; wired their IDs into `.env` | [Figures KB](docs/SCREENSHOTS.md#1-data-foundation--knowledge-bases) |
| 20:11 → 20:26 | Three further deployment iterations | `deploy-20261020-*.log` (3 files) |
| 20:31 → 20:38 | Invoked the deployed runtime two ways (project `invoke` command, then `agentcore invoke`) | [Figure 11](screenshots/Script_test.png), [Figure 12](screenshots/agentcore_cli_test.png) |
| 20:42 → 20:56 | Ran all three end-to-end scenarios | [Figures 4–10](docs/SCREENSHOTS.md#2-end-to-end-scenario--return-request) |
| 21:00 → 21:21 | Ran the automated test suite in slices | [Figures 2–3](docs/SCREENSHOTS.md#6-automated-evaluation) |
| 21:33 | Captured the X-Ray service map | [Figure 13](screenshots/xray_service_map.png) |

## 3. What I ran, and what happened

### Scenario 1 — return request
**Route:** Orchestrator → Inventory → Refund → Communication. Customer CUST-001 asks to return the wireless headphones from order ORD-27176.

- Inventory finds a **delivered** order dated 8 October 2026 for a **Premium** customer. State `0 → 1` (observed 15.0 s).
- Refund applies the **60-day Premium window** and approves. State `1 → 2` (15.9 s).
- Communication writes the customer reply using the full workflow record. State `2 → 3` (14.6 s).
- **Caveat:** in the captured run the RefundAgent asked for a return reason instead of calling `initiate_refund`, so the DynamoDB record update was **not** exercised. See §5.

[Figure 4](screenshots/scenario_tests_step_01.png) · [Figure 5](screenshots/scenario_tests_step_02.png) · [Figure 6 (alternate crop)](screenshots/scenario_tests_step_03_1.png)

### Scenario 2 — policy question
**Route:** Orchestrator → Policy (3 retrievers in parallel) → Communication. Customer CUST-002 asks: *"What is the return policy for premium customers?"* Inventory and Refund are skipped.

- `search_all_policies` starts the Returns, Shipping and Warranty retrievers together; all three respond in about **14 seconds**.
- The PolicyAgent step completes in **25.8 s** including retrieval.
- Communication answers with the 60-day window, qualification criteria, return process and Premium benefits (13.1 s).

[Figure 7](screenshots/scenario_tests_step_04_3.png) · [Figure 8](screenshots/scenario_tests_step_04_4.png) · [Figure 9](screenshots/scenario_tests_step_05_1.png)

### Scenario 3 — calculation
**Route:** Orchestrator → Communication only. Customer CUST-003 asks: *"How much would 5 items at $29.99 be with a 10% discount?"*

- No Inventory, Policy or Refund step runs; the guardrail does **not** block the calculation.
- One state write, `0 → 1` (11.3 s).
- The reply reports **$134.95**; see the rounding caveat in §5.

[Figure 10](screenshots/scenario_tests_step_06.png)

### Deployed runtime
The same policy question was sent to the live AgentCore Runtime two ways: the project's own `invoke` command and the AgentCore CLI. Both returned a grounded reply and an X-Ray trace ID; the CLI call also returned a resumable session ID.

[Figure 11](screenshots/Script_test.png) · [Figure 12](screenshots/agentcore_cli_test.png)

### Automated checks
The course suite checks the agents, guardrail, memory and Knowledge Bases; every check reads real AWS state. The capture in the journal shows **30 checks across Tasks 2–5, all passing**. The final slice retained in `screenshots/` shows **120/120 points (100%)** and includes the Task 6 observability checks.

[Figure 2](screenshots/test_agent_tests_01.png) · [Figure 3](screenshots/test_agent_tests_03.png) · [Final score](screenshots/test_agent_tests_04.png)

> **Note on Task 6:** the PDF narrative doesn't include the Task 6 output, but [`test_agent_tests_04.png`](screenshots/test_agent_tests_04.png) does show those checks and the 120/120 score. Both accounts are kept here so the difference is clear.

## 4. Design decisions I made

| Decision | Why |
|---|---|
| Haiku 4.5 at temperature 0.0 for routing; Sonnet 4.5 for workers | Cheap, repeatable routing; reasoning capacity where it is needed. |
| Facts kept apart from decisions | Inventory only reports database facts; Refund applies the tier window. One agent cannot guess eligibility from partial data. |
| One agent speaks to the customer | The orchestrator never writes the final answer; its last tool call is always Communication, so tone is controlled in one place. |
| Workers share state, not transcripts | Each agent writes its output into one DynamoDB record instead of passing long conversation histories. |
| Deterministic retrieval, warm writing | Retrievers run at 0.0; Communication at 0.3. |
| Guardrail topic boundary | "Pricing negotiations" means haggling or changing an advertised price, and explicitly **allows arithmetic applying a stated discount** — otherwise a question like *"5 items at $29.99 with 10% off?"* could be blocked. |

## 5. Observations and next steps I recorded

These are honest limits of the captured runs, not claims of success.

1. **Latency is in tens of seconds.** Worker steps took 11–26 s; the return flow runs three workers in sequence (~45 s of worker time). Streaming the reply and running independent steps concurrently would come first. The calculation scenario skipping three workers is the clearest example of routing paying off.
2. **Live parallel output still interleaves.** The trace helper suppresses sub-agent output during the parallel step, yet streamed text still mixes in the live view ([Figure 14](screenshots/scenario_tests_step_04_2.png)). The clean results after `[END]` and the final answers were unaffected.
3. **Retrievers overlap.** `customer_tiers.txt` sits under all three S3 prefixes, so every retriever can answer a Premium-tier question and outputs partly duplicate each other. In one run the Warranty retriever listed a 2-year electronics warranty beside the 1-year Standard figure from the others; the final answer used 1 year. Cleaner source documents and explicit conflict reconciliation would tighten grounding.
4. **The refund record update was not exercised.** Approval happened, but `initiate_refund` was not called in the captured run. A prompt change or an explicit confirmation step would make the approval → record-update hand-off clear.
5. **Rounding is ambiguous.** The reply reports $134.95 by rounding the $14.995 discount to $15.00 before subtracting. Rounding the final total gives $134.96. Stating a rounding rule in the prompt would remove the ambiguity.

**Still to try:** guardrail blocks for the negotiation / competitor / legal-threat examples, and memory recall across sessions. The Task 6 output isn't in the journal figures, though it is in `screenshots/test_agent_tests_04.png`.

## 6. Commands used

```bash
python config.py                                        # bucket names for the Knowledge Bases
python infrastructure/seed_data.py                      # seed DynamoDB + upload policy documents
# create three Knowledge Bases in the Bedrock console, sync, add IDs to .env
python src/agent_orchestrator.py deploy                 # guardrail, runtime, memory, observability
python tests/test_agent.py all                          # automated checks
python src/agent_orchestrator.py test                   # three scripted scenarios
python src/agent_orchestrator.py invoke "What is the return policy for premium customers?"
agentcore invoke "What is the return policy for premium customers?"
python infrastructure/cleanup.py --yes                  # delete everything the project created
```

## 7. Figure index

Each PDF figure maps to a retained screenshot. See the full [screenshot index](docs/SCREENSHOTS.md) for descriptions.

| Figure | Screenshot |
|---|---|
| 1 — Seed data | [`seeding_data.png`](screenshots/seeding_data.png) |
| 2 — Tests, Tasks 2–5 | [`test_agent_tests_01.png`](screenshots/test_agent_tests_01.png) |
| 3 — Parallel retrieval check | [`test_agent_tests_03.png`](screenshots/test_agent_tests_03.png) |
| 4 — Inventory step | [`scenario_tests_step_01.png`](screenshots/scenario_tests_step_01.png) |
| 5 — Refund step | [`scenario_tests_step_02.png`](screenshots/scenario_tests_step_02.png) |
| 6 — Communication step | [`scenario_tests_step_03_1.png`](screenshots/scenario_tests_step_03_1.png) (same step, different crop) |
| 7 — Retriever outputs | [`scenario_tests_step_04_3.png`](screenshots/scenario_tests_step_04_3.png) |
| 8 — Policy step completion | [`scenario_tests_step_04_4.png`](screenshots/scenario_tests_step_04_4.png) |
| 9 — Policy reply | [`scenario_tests_step_05_1.png`](screenshots/scenario_tests_step_05_1.png) |
| 10 — Math scenario | [`scenario_tests_step_06.png`](screenshots/scenario_tests_step_06.png) |
| 11 — Project invoke | [`Script_test.png`](screenshots/Script_test.png) |
| 12 — AgentCore CLI invoke | [`agentcore_cli_test.png`](screenshots/agentcore_cli_test.png) |
| 13 — X-Ray map | [`xray_service_map.png`](screenshots/xray_service_map.png) |
| 14 — Interleaved output | [`scenario_tests_step_04_2.png`](screenshots/scenario_tests_step_04_2.png) |
