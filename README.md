# AquaSentinel — n8n Workflows

n8n workflow exports for the AquaSentinel water-quality prototype. The event
orchestrator runs in n8n: it receives anomaly events and user queries, invokes a
real AI agent (Sarvam `sarvam-105b`) that gathers evidence through HTTP tools,
then updates the mock backend event.

> These are **workflow exports** (JSON), not a full n8n install. Import them into
> an n8n instance that can reach your mock backend at `http://host.docker.internal:8000`.

## Workflows

| File | Workflow | Role |
| --- | --- | --- |
| `console-ai-agent.json` | **AquaSentinel — AI Agent Console** | Primary. Manual trigger → event/query input → priority queue → real AI Agent node with 4 tools → event update. |
| `console-priority-queue.json` | AquaSentinel — Console & Priority Queue | Superseded iteration of the queue (item-stream version). |
| `unified-to-agent.json` | AquaSentinel — Unified Event & Query | Webhook-based unified event + query path (older). |
| `anomaly-to-agent.json` | AquaSentinel — Event Investigation | Webhook event → agent path (older). |
| `user-query-to-agent.json` | AquaSentinel — User Query | Webhook user-query → agent path (older). |

## Primary workflow (`console-ai-agent.json`)

```
Manual Trigger
  → Event Input                      (code node — pick a scenario to emit)
  → Sort & Filter (Queue)            (priority: CRITICAL > HIGH > MEDIUM > LOW; queries last)
  → Has Work?                        (IF: any work left? else Queue Empty)
  → Is Query?                        (IF: user query vs anomaly event)
       ├─ true  → Prepare Agent Prompt
       └─ false → Create Event (backend) → Prepare Agent Prompt
  → AquaSentinel Agent               (native AI Agent node, Sarvam model + 4 tools)
  → Parse Agent Result
  → Is User Query?                   (IF: did we investigate an event or answer a query?)
       ├─ true  → Query Answered
       └─ false → Update Event (PATCH) → Query Answered
```

Nodes:
- **Event Input** — code node; edit `eventsToEmit` to pick a scenario
  (`salt`, `silt`, `sensor-spike`, `sensor-offline`, `wastewater`, `prediction`)
  or set `query` to ask the agent a question.
- **AquaSentinel Agent** — n8n AI Agent node (typeVersion 2) using the Sarvam
  chat model with four HTTP Request tools: `get_current_readings`,
  `get_historical_readings`, `get_anomaly_event`, `get_sensor_status`.
- **Create Event / Update Event** — HTTP calls to the mock backend
  (`POST /api/events`, `PATCH /api/events/{event_id}`).

## Import

1. In n8n, go to **Workflows → ⋮ → Import from File**.
2. Select the `.json` export you want.
3. For `console-ai-agent.json`, connect the **Sarvam Chat Model** node to your
   OpenAI-compatible Sarvam credential (base URL `https://api.sarvam.ai/v1`,
   model `sarvam-105b`, and set **Responses API** to **off**).
4. Ensure the mock backend is reachable at `http://host.docker.internal:8000`
   (update the HTTP nodes if your backend lives elsewhere).

## Run

Open the workflow, edit **Event Input** to pick a scenario, then press
**Execute Workflow** on the **Manual Trigger** node. Per-node output is visible
in the editor after the run.

## Notes

- The agent is event-triggered: it is not called for every sensor sample, and
  the `Queue Empty` path exits without invoking the LLM.
- Agent responses update a specific `event_id`, never "the latest event".
- API keys and credentials are not part of these exports.