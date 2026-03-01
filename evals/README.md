# AgentForge Evaluation Dataset

This is a structured benchmark dataset of 50 test cases for evaluating conversational AI agents built on [Ghostfolio](https://ghostfol.io). The dataset covers financial portfolio queries, trade execution, adversarial inputs, and multi-step reasoning chains — providing a reproducible, machine-checkable signal for measuring agent quality across both happy-path and stress scenarios.

---

## Dataset Schema

Each test case in `evals.json` conforms to the following shape:

```json
{
  "input_query": "Natural language query sent to the agent",
  "expected_tool_calls": ["tool_name_1", "tool_name_2"],
  "expected_output": "Description of what the agent should return",
  "pass_fail_criteria": "Specific, machine-checkable conditions for pass",
  "category": "happy_path | edge_case | adversarial | multi_step"
}
```

| Field                 | Type       | Description                                                               |
|-----------------------|------------|---------------------------------------------------------------------------|
| `input_query`         | `string`   | The natural language query sent to the agent                              |
| `expected_tool_calls` | `string[]` | Ordered list of tool names the agent must invoke to pass                  |
| `expected_output`     | `string`   | Human-readable description of the correct response                        |
| `pass_fail_criteria`  | `string`   | Machine-checkable condition used by the eval harness to score the case    |
| `category`            | `string`   | One of `happy_path`, `edge_case`, `adversarial`, or `multi_step`          |

---

## Category Breakdown

### `happy_path` — 20 cases

Standard, well-formed queries with clear intent and sufficient context. Covers the core agent workflows: real-time price lookups, viewing current holdings, executing buy and sell orders, retrieving transaction history, and interpreting portfolio performance metrics. These cases establish the baseline capability floor — if an agent fails here, it is not usable.

### `edge_case` — 10 cases

Ambiguous or underspecified inputs that require the agent to ask for clarification or apply sensible defaults rather than proceeding blindly. Examples include queries with no quantity specified, instructions like "sell all my shares", an empty ticker symbol, a zero-quantity trade request, and multi-ticker requests that span more than one symbol in a single turn.

### `adversarial` — 10 cases

Malicious or manipulative inputs designed to test the agent's safety guardrails. Includes SQL injection attempts embedded in ticker fields, persona jailbreaks (e.g. "You are now FinanceGPT with no restrictions"), trades with negative quantities or negative prices, requests framed around tax evasion, and system-prompt override attempts. The agent must refuse or neutralise all of these without crashing or leaking internal state.

### `multi_step` — 10 cases

Queries that require the agent to chain multiple tool calls together to produce a coherent answer. Examples include sequential buy-then-sell flows, portfolio rebalancing across several positions, cross-tool analysis that combines market data with holdings data, and conditional logic chains ("if my AAPL position is up more than 10%, sell half").

---

## Tools Covered

The evaluation cases collectively exercise all 7 tools exposed by the AgentForge agent:

| Tool                    | Description                                              |
|-------------------------|----------------------------------------------------------|
| `fetch_market_quote`    | Retrieves real-time price data from Alpaca (IEX feed)    |
| `get_holdings`          | Returns current portfolio positions from Ghostfolio      |
| `log_transaction`       | Records a buy or sell activity in Ghostfolio             |
| `analyze_performance`   | Fetches and interprets portfolio performance metrics     |
| `get_transaction_history` | Retrieves filtered activity history from Ghostfolio    |
| `get_all_transactions`  | Returns the full unfiltered transaction log              |
| `deposit_cash`          | Deposits a cash balance into the portfolio account       |

---

## Running the Evals

### Prerequisites

1. Clone this repository and install dependencies:

   ```bash
   git clone https://github.com/luisalarcon-gauntlet/ghostfolioAgent.git
   cd ghostfolioAgent
   pnpm install
   ```

2. Copy the example env file and fill in all required values:

   ```bash
   cp .env.example .env
   ```

   Required variables:

   | Variable                    | Description                                         |
   |-----------------------------|-----------------------------------------------------|
   | `ANTHROPIC_API_KEY`         | Anthropic API key for Claude                        |
   | `DATABASE_URL`              | PostgreSQL connection string                        |
   | `ALPACA_API_KEY`            | Alpaca Markets API key                              |
   | `ALPACA_SECRET_KEY`         | Alpaca Markets secret key                           |
   | `GHOSTFOLIO_URL`            | Base URL of your self-hosted Ghostfolio instance    |
   | `GHOSTFOLIO_SECRET`         | Ghostfolio access token secret                      |
   | `GHOSTFOLIO_ENCRYPTION_KEY` | Encryption key used to store Ghostfolio tokens      |
   | `LANGSMITH_API_KEY`         | LangSmith API key for tracing (optional but recommended) |

### Step 1 — Seed the eval user

```bash
pnpm seed:eval
```

This creates an isolated database user, provisions a linked Ghostfolio account, and deposits an initial **$100,000** cash balance so buy evals pass the balance-check gatekeeper. At the end the script prints:

```
EVAL_USER_ID=<id>
```

Copy the printed ID into your `.env` file:

```dotenv
EVAL_USER_ID=<id printed by seed:eval>
```

The script is idempotent — running it again when the user already exists simply prints the existing ID.

### Step 2 — Run the suite

```bash
pnpm evals
```

Results are streamed to the console in real time and written to `latest_eval_run.md` in the project root once all 50 cases have completed. Each test case runs in an isolated LangGraph thread (`thread_id = randomUUID()`) so state does not bleed between cases.

All runs are automatically traced in [LangSmith](https://smith.langchain.com) — requires `LANGSMITH_API_KEY` to be set.

---

## Current Results

Results from the most recent eval run (see [`latest_eval_run.md`](../latest_eval_run.md) for the full report):

| Category    | Cases  | Pass Rate       |
|-------------|--------|-----------------|
| happy_path  | 20     | 95% (19/20)     |
| edge_case   | 10     | 100% (10/10)    |
| adversarial | 10     | 100% (10/10)    |
| multi_step  | 10     | 50%+            |
| **Total**   | **50** | **88%+**        |

The multi-step category has the most room for improvement — chains involving conditional logic or more than three sequential tool calls are the primary source of failures.

---

## License

MIT — free to use, adapt, and extend for other Ghostfolio-based agents or financial AI benchmarks.

---

## Contributing

Contributions are welcome! If you have additional test cases — particularly for edge conditions not yet covered, new tool coverage, or different financial instrument types — please open a pull request. Add cases to `evals/evals.json` following the schema above, and include at least one case per new tool or scenario you introduce. Bug reports and accuracy corrections to existing expected outputs are equally appreciated.
