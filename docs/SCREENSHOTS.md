# Evidence Index

Proof-of-work screenshots from the completed NovaMart project. All files are in [`Submission/`](../Submission/). The portfolio [README](../README.md) links to a curated subset; this page is the complete index.

Screenshots are grouped by the capability they demonstrate. Several scenario screenshots are sequential slices of the same terminal run, so they are listed in execution order. The journal's Figures 1–14 map to these files in [Build Journal §7](../BUILD_JOURNAL.md#7-figure-index).

Notes on what the evidence does **not** show are collected at the end of this page.

---

## 1. Data foundation & knowledge bases

| File | What it shows |
|---|---|
| [`seeding_data.png`](../Submission/seeding_data.png) | `seed_data.py` inserting 4 customers and 11 orders and uploading 6 policy documents to S3 — then "Seeding complete!". |
| [`knowledge_bases.png`](../Submission/knowledge_bases.png) | Bedrock console listing all three knowledge bases as **Available**: `novamart-returns-policy-kb`, `novamart-shipping-policy-kb`, `novamart-warranty-policy-kb`. |
| [`returns_kb.png`](../Submission/returns_kb.png) | Returns KB detail: ID `AXNIG9KRYW`, RAG type *vector store*, one available S3 data source `returns-docs`. |
| [`shipping_kb.png`](../Submission/shipping_kb.png) | Shipping KB detail: ID `KICKZWZEJQ`, "Sync completed for data source – shipping-docs". |
| [`warranty_kb.png`](../Submission/warranty_kb.png) | Warranty KB detail: ID `PJS1IUYRXD`, "Sync completed for data source – warranty-docs". |

## 2. End-to-end scenario — return request

Customer CUST-001 (Premium) asks to return order ORD-27176.

| File | What it shows |
|---|---|
| [`scenario_tests_step_01.png`](../Submission/scenario_tests_step_01.png) | Orchestrator initialises the session and routes to the **Inventory Agent**: order status `Delivered`, product, price $149.99, and customer tier `Premium`; state version `0 → 1`. |
| [`scenario_tests_step_02.png`](../Submission/scenario_tests_step_02.png) | **Refund Agent** reads inventory context, applies the 60-day Premium window and approves, then asks the customer for a return reason; state version `1 → 2`. It does **not** call `initiate_refund` in this run. |
| [`scenario_tests_step_03_1.png`](../Submission/scenario_tests_step_03_1.png) | **Communication Agent** composes the final response: approval confirmed, prepaid label, $149.99 refund in 5–7 business days; state version `2 → 3`. |
| [`scenario_tests_step_03_2.png`](../Submission/scenario_tests_step_03_2.png) | End-of-run summary, "X-Ray trace … published successfully", and the next scenario beginning. |

## 3. End-to-end scenario — policy question

Customer CUST-002 asks about the return policy for Premium customers; the policy agent fans out to all three KBs.

| File | What it shows |
|---|---|
| [`scenario_tests_step_04_1.png`](../Submission/scenario_tests_step_04_1.png) | **Policy Agent** starts `search_all_policies`; `[PARALLEL RETRIEVAL – START]` spawns Returns, Shipping and Warranty retrievers with their KB IDs. |
| [`scenario_tests_step_04_2.png`](../Submission/scenario_tests_step_04_2.png) | Continued trace — the three retriever sub-agents querying their respective knowledge bases. Live streamed text still interleaves here (journal Figure 14); the printed results after `[END]` are clean. |
| [`scenario_tests_step_04_3.png`](../Submission/scenario_tests_step_04_3.png) | Continued trace — retrieved policy passages from the knowledge bases. |
| [`scenario_tests_step_04_4.png`](../Submission/scenario_tests_step_04_4.png) | Continued trace — results returning from the parallel sub-agents. |
| [`scenario_tests_step_04_5.png`](../Submission/scenario_tests_step_04_5.png) | Tail of the parallel section — `[PARALLEL RETRIEVAL – END]` and combined results handed to the coordinator. |
| [`scenario_tests_step_05_1.png`](../Submission/scenario_tests_step_05_1.png) | **Communication Agent** delivers the grounded answer: 60-day Premium window, qualification criteria, return process, 3-year electronics warranty and other benefits. |
| [`scenario_tests_step_05_2.png`](../Submission/scenario_tests_step_05_2.png) | Continuation of the final customer-facing response. |
| [`scenario_tests_step_05_3.png`](../Submission/scenario_tests_step_05_3.png) | Tail of the policy scenario run. |

## 4. End-to-end scenario — pricing calculation

| File | What it shows |
|---|---|
| [`scenario_tests_step_06.png`](../Submission/scenario_tests_step_06.png) | **Communication Agent** presents the arithmetic: 5 items × $29.99 = $149.95, 10% discount = −$15.00, **final total $134.95** — plus the published X-Ray trace ID. Confirms the guardrail does not block legitimate discount arithmetic. Caveat: the discount is rounded before subtracting; rounding the total instead gives $134.96. |

## 5. Deployed runtime invocations

| File | What it shows |
|---|---|
| [`Script_test.png`](../Submission/Script_test.png) | `python src/agent_orchestrator.py invoke "…"` against the live AgentCore Runtime ARN — session ID, full policy answer and X-Ray trace ID. |
| [`agentcore_cli_test.png`](../Submission/agentcore_cli_test.png) | `agentcore invoke "…"` — the deployed agent returning the grounded 60-day Premium policy answer, session ID and invocation log path. |

## 6. Automated evaluation

Four sequential slices of one `python tests/test_agent.py all` run — the checks read real AWS state. The journal narrative counts 30 checks across Tasks 2–5; the final slice below also shows Task 6 and the 120/120 score.

| File | What it shows |
|---|---|
| [`test_agent_tests_01.png`](../Submission/test_agent_tests_01.png) | Task 2: all agent-instantiation, tool-count, model-tier and `ThreadPoolExecutor` checks pass. Task 3: guardrail policies/version and READY PUBLIC/HTTP runtime pass. Task 4: ACTIVE memory with summary strategy and 7-day expiry pass. Task 5 checks begin. |
| [`test_agent_tests_02.png`](../Submission/test_agent_tests_02.png) | Task 5 parallel retrieval: `[PARALLEL RETRIEVAL – START]`, three sub-agents dispatched concurrently via `ThreadPoolExecutor`. |
| [`test_agent_tests_03.png`](../Submission/test_agent_tests_03.png) | `[PARALLEL RETRIEVAL – END] — All 3 KBs responded (14.4s)` with each retriever's grounded passages, and the combined-results check passing. |
| [`test_agent_tests_04.png`](../Submission/test_agent_tests_04.png) | Task 6: CloudWatch INFO logging and 100% X-Ray sampling pass; 5 NovaMart traces detected; **Score: 120/120 pts (100%) — "Perfect score! All tasks complete."** |

## 7. Observability

| File | What it shows |
|---|---|
| [`xray_service_map.png`](../Submission/xray_service_map.png) | CloudWatch/X-Ray service map: `Client → NovaMart-Orchestrator → {RefundAgent, PolicyAgent, Knowledge_Retrieval, InventoryAgent, CommunicationAgent}` — the agent call chain captured as distributed traces. |

## 8. Delivered code

| File | What it shows |
|---|---|
| [`agent_orchestrator.py`](../Submission/agent_orchestrator.py) | My implementation: five agents and their tools, guardrail creation, AgentCore deployment pipeline and HTTP entry point. It uses the course-provided WorkflowState helpers with optimistic locking. |
| [`config.py`](../Submission/config.py) | Provided by the course: lazy, cached configuration resolving resource IDs from CloudFormation exports and environment variables. |
| `.env` | Local runtime configuration — **intentionally git-ignored**. A placeholder template is provided as [`.env.example`](../Submission/.env.example). |

## 9. Not captured — therefore not claimed

From the build journal's own observations:

- Guardrail blocking the negotiation, competitor and legal-threat examples.
- AgentCore Memory recalling a previous session.
- The Task 6 observability test output **in the journal figures** (it is present in [`test_agent_tests_04.png`](../Submission/test_agent_tests_04.png) in `Submission/`).
- A successful `initiate_refund` call updating the DynamoDB record after approval.
- A guardrail block decision for any denied topic.
