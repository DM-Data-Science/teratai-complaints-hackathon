# The API your agent must expose

Every team exposes the same small HTTP API, so the live round treats every team the same way. JSON in, JSON out. Use any framework (FastAPI, Flask, Express…); [`build_guide.md`](build_guide.md#the-api) has a working FastAPI example.

Test yours at any time with the same script the organisers use:

```bash
python scoring/api_check.py http://localhost:8000
```

## Endpoints

### `POST /complaints`

Hand the agent one new complaint. The body is the six fields of a `complaints.csv` row:

```json
{
  "complaint_id": "CMP-00244",
  "customer_id": "CUS-02394",
  "received_at": "2026-07-20 22:32",
  "channel": "Email",
  "subject": "ATM did not dispense cash",
  "text": "Hello, the machine did not dispense all of the cash when I withdrew about RM1,500 last Friday, but the whole amount was deducted. Please resolve this quickly."
}
```

- Return `202` (or `200`/`201`) with `{"complaint_id": "...", "status": "processing"}` and process in the background. You may also decide synchronously and return `"status": "done"`, but answer within 120 seconds.
- Missing or empty required fields (`complaint_id`, `customer_id`, `received_at`, `text`): return `400` or `422`, never a `5xx`.
- The same `complaint_id` posted again must not create a second case or change the decision: return the existing case's status (`200` or `409`).
- A `customer_id` that isn't in the data must not crash the service. Reject it with a `4xx`, or accept it and flag the case.
- Use `received_at` from the complaint, not today's date, for "yesterday", "last Friday" and so on.

### `GET /cases/{complaint_id}`

Return the case. `404` if you have never seen that ID.

While the agent is working:

```json
{"complaint_id": "CMP-00244", "status": "processing"}
```

When it has decided (`"status": "done"`), every field below is required:

```json
{
  "complaint_id": "CMP-00244",
  "status": "done",
  "category": "ATM_NOT_DISPENSED",
  "matched_txn_id": "TXN-0088211",
  "outcome": "REFUND",
  "refund_amount_myr": "550.00",
  "priority": "P2",
  "reply_language": "EN",
  "route": "HUMAN_APPROVAL",
  "reply_text": "Dear customer, ...",
  "self_check": "passed",
  "case_state": "awaiting_approval"
}
```

| Field | Values |
|---|---|
| `status` | `processing`, `done`, or `error` (the agent failed; say why in an `error` field) |
| `category`, `matched_txn_id`, `outcome`, `refund_amount_myr`, `priority`, `reply_language`, `route`, `reply_text` | As in `predictions.csv` ([`scoring.md`](scoring.md)). `matched_txn_id` is `""` when there is none |
| `self_check` | `passed` or `flagged`: the result of your code check that every transaction ID and amount in the reply is in the evidence |
| `case_state` | Optional, free text: where the case is in your workflow (`auto_resolved`, `awaiting_approval`, `with_fraud_team`, `approved`, `sent`…) |

These fields are the **agent's own decision**. A human approving, editing or rejecting the case later must not change them; keep human actions separately.

### `GET /health`

Return `200` when the service is up.

## Being reachable in the live round

The organisers' laptop calls your API during your demo slot. Either:
- run it on your laptop on the venue Wi-Fi, listening on all interfaces (for example `uvicorn app:app --host 0.0.0.0 --port 8000`), and give us `http://<your-laptop-ip>:8000`; or
- put it behind a tunnel (ngrok, Cloudflare Tunnel) and give us that URL.

We check that we can reach you at Checkpoint 2 on Day 2. Keep API keys for your LLM on your side; our calls carry no credentials.

## What the organisers' test checks

`scoring/api_check.py` sends all its complaints at once, polls each case until it is `done` (120 seconds allowed per complaint), checks every field against this contract, and scores the answers exactly like the leaderboard. It also runs five robustness probes: `/health`, an unknown case ID (`404`), a complaint without text (`4xx`), an unknown customer (no `5xx`), and the same complaint posted twice (no change, no duplicate). Probe results and timing go to the judges as part of the engineering score.
