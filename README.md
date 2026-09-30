# Trading Agents Lite

**English** | [Bahasa Indonesia](README.id.md)

A lite version of [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents), rebuilt as a **Claude Agent Skill**. It runs the same multi-agent research workflow (analysts, a bull vs bear debate, a trader, a risk committee and a portfolio manager) directly inside Claude, so you can use it with a Claude subscription instead of a separate API key.

> This is a research and learning tool. It does not give financial advice, it does not place trades, and it makes no claim about profitability.

---

## Contents

- [What this is](#what-this-is)
- [How it works](#how-it-works)
- [Lite vs the original](#lite-vs-the-original)
- [Strengths](#strengths)
- [Limitations](#limitations)
- [What this project does not claim](#what-this-project-does-not-claim)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Running the scripts on their own](#running-the-scripts-on-their-own)
- [Repository structure](#repository-structure)
- [Credits](#credits)
- [Disclaimer](#disclaimer)

---

## What this is

The original TradingAgents is a Python framework built on LangGraph. Every agent is a separate call to a language model through an API, so you need API keys from a provider such as OpenAI, Anthropic or Google and you pay per token.

Trading Agents Lite keeps the **role design** of the original and moves it into a Claude skill: a set of Markdown instructions plus a few small Python helper scripts. Claude reads the instructions and plays the roles, while the scripts handle the parts that must be exact (indicator math, price rounding, position sizing, decision memory).

It is called "lite" because it deliberately drops parts of the original: there is no graph engine, no multi-provider support, no checkpoint recovery and no programmatic batch runs. In return, it needs almost no setup.

This project is an independent reimplementation. It contains no code from the original repository and is not affiliated with or endorsed by Tauric Research or Anthropic.

## How it works

One analysis covers one ticker on one analysis date and passes through seven stages. Every role writes its output to its own file in a run folder, so later roles read files instead of relying on what was said earlier in the conversation.

1. **Data.** A data snapshot is built from one or more sources: `fetch_data.py` (Yahoo Finance through `yfinance`, when the network allows it), a price CSV you upload, or web search. Nothing dated after the analysis date is used. Indicators are computed by `indicators.py`, not estimated by the model.
2. **Analyst team.** Four analysts write independent reports: Technical, Fundamentals, News & Macro, and Sentiment. None of them sees the others' reports.
3. **Research team.** A Bull researcher and a Bear researcher debate for 1 to 3 rounds. Each turn must first rebut the opponent's strongest point. A Research Manager judges the debate and writes an investment plan with a rating.
4. **Trader.** The plan becomes a concrete proposal. `risk_tools.py` calculates entry, stop, targets, reward to risk and position size, rounded to valid price ticks and, for Indonesian stocks, whole lots.
5. **Risk committee.** Aggressive, Neutral and Conservative members challenge the proposal and each proposes a concrete adjustment.
6. **Portfolio Manager.** Accepts or rejects each adjustment, applies lessons from earlier decisions, and gives the final rating on a five-tier scale: Buy, Overweight, Hold, Underweight, Sell.
7. **Outputs.** A short summary in the chat, a full report as a Markdown file, and an entry in the decision log (`trading_agents_log.jsonl`) that serves as memory for the next analysis.

The skill runs in two modes and picks one automatically:

- **Subagent mode (Claude Code, Cowork):** each role runs as a separate subagent with its own context. This is the closest to the original design.
- **Single-context mode (claude.ai chat):** one Claude conversation plays all roles in sequence. Separation is enforced by procedure: each report is written only from its own input files and is not revised after later reports appear.

Depth is adjustable: `quick`, `standard` (default) or `deep`. Deeper runs use longer reports and more debate rounds.

![Agent flow of Trading Agents Lite](assets/diagram.png)

## Lite vs the original

| Aspect | TradingAgents (original) | Trading Agents Lite |
|---|---|---|
| Form | Python framework on LangGraph with a CLI and a Python API | Claude skill: Markdown instructions plus helper Python scripts |
| Language models | Many providers through API keys, including local models via Ollama | Claude only, whichever Claude model you are using |
| Cost model | Pay per API token (or free with local models) | Counts toward your Claude plan usage, no separate API key |
| Agent separation | Each agent is a separate model call in a graph | Separate subagents in Claude Code; one shared context with procedural separation in claude.ai |
| Model mixing | Different models for "deep thinking" and "quick thinking" roles | One model for all roles |
| Market data | Built-in data vendors (Yahoo Finance, Alpha Vantage and others in recent versions) | `yfinance` script where the network allows it, otherwise uploaded CSV or web search |
| Memory | Automatic decision log with reflections | `decision_log.py`; on claude.ai you keep and re-upload the log file yourself |
| Crash recovery | Checkpoint resume | Not available |
| Automation | Scriptable across many tickers and dates | Conversational, one analysis at a time |
| Setup | Install the package and configure API keys | Upload a zip (claude.ai) or copy a folder (Claude Code) |
| Market rules | Works with any market Yahoo Finance covers | Also adds explicit Indonesia Stock Exchange rules: price ticks, lots, notes on auto rejection limits |

## Strengths

- **No API key needed.** The analysis runs inside Claude and uses your existing plan. This was the main reason for building the lite version.
- **Computed numbers.** RSI, MACD, ADX, ATR, Bollinger Bands, moving averages, pivot levels, volatility, drawdown and beta come from `indicators.py`. The agents are instructed to copy numbers from script output or cited sources and to write "not available" instead of guessing.
- **Traceable sources.** Web findings are logged in `sources.md` with date, publisher and URL, and reports cite them.
- **Look-ahead protection.** Prices are cut off at the analysis date, quarterly statements are only used about 45 days after the period ends (to approximate reporting delay), and headlines published after the date are dropped. This makes it possible to analyse a past date, with the caveats listed below.
- **Exchange-valid trade plans.** Stops and targets are rounded to valid price ticks and sizes to whole lots on the Indonesia Stock Exchange, and positions are sized from a risk budget.
- **Flexible data input.** It still works when Yahoo Finance is unreachable. The CSV parser reads exports from Yahoo Finance, Investing.com (including Indonesian headers and number formats such as `9.875` for 9875), Nasdaq.com, TradingView and broker apps.
- **Transparency.** Every role's output is saved and assembled into a full report, so you can see how the final rating was reached and where you disagree.
- **Lightweight.** No graph framework. The scripts need only `pandas` and `numpy`; `yfinance` is optional.

## Limitations

Please read these before relying on any output.

1. **One model plays every role.** All agents use the same Claude model, so they share its tendencies. In claude.ai they also share one conversation, which means independence between agents is procedural, not structural. The bull vs bear debate can be less adversarial than debates between separately prompted models.
2. **No model choice.** You cannot assign a stronger model to the decision roles and a cheaper one to the rest, as the original allows. Output quality follows whichever Claude model you use.
3. **Data is thinner on claude.ai.** The claude.ai sandbox blocks Yahoo Finance by default. Without an uploaded CSV, technical analysis falls back to web data and is clearly less complete. Fundamentals for many Indonesian stocks are uneven on free sources. Sentiment is a qualitative reading of search results, not a systematic feed from StockTwits, Stockbit or X.
4. **Usage limits.** A standard analysis produces many long messages. Deep mode and multi-ticker comparisons use noticeably more of your plan and can hit usage limits.
5. **Results vary between runs.** Language model output is not deterministic. The same ticker and date can produce different reports and occasionally a different rating.
6. **Not validated.** There is no systematic backtest behind this skill. The helper scripts were checked on synthetic data in several file formats, but the full workflow has not been evaluated on real market history. Indicator values can differ slightly from charting platforms because of smoothing and seeding choices.
7. **Heuristics are simplifications.** The trend score, pivot detection, liquidity threshold and the 45-day reporting lag are rules of thumb, not precise models.
8. **Past dates are only partly protected.** Company profile ratios from Yahoo are current values, and web or social sources cannot always be filtered to what was known on a past date. The skill flags this, but backtest mode is weaker than a proper historical database.
9. **Market rules change.** Price tick tables and auto rejection limits on the Indonesia Stock Exchange can change. The skill asks Claude to verify current rules when they matter.
10. **No automation, recovery or execution.** There is no scheduling, no checkpoint resume, and no connection to any broker.
11. **Manual memory on claude.ai.** Files in the claude.ai sandbox do not persist between chats. To keep the decision history, download `trading_agents_log.jsonl` after an analysis and upload it again next time.

## What this project does not claim

- It does not claim to beat the market or to reproduce any returns reported for the original framework.
- It does not claim that its ratings are correct or suitable for your situation.
- It does not claim to be equivalent to the original TradingAgents. It is a simplified adaptation of its role design.

## Requirements

**For claude.ai (web or desktop app)**
- A Claude plan that supports custom skills, with code execution enabled. See the official documentation on [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) and the [Claude Help Center](https://support.claude.com) for the current plans and settings.

**For Claude Code**
- Claude Code installed, see the [Claude Code documentation](https://docs.claude.com/en/docs/claude-code/overview).
- Python 3.10 or newer with the packages in `requirements.txt`.

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/trading-agents-lite.git
cd trading-agents-lite
```

Replace `<your-username>` with the account that hosts this repository. You can also use **Code > Download ZIP** on the GitHub page and extract it.

### 2a. Install on claude.ai

1. Make sure code execution is enabled in Claude's settings.
2. Create a zip file that contains the `trading-agents` folder at its root (the folder that holds `SKILL.md`):

   macOS or Linux:
   ```bash
   zip -r trading-agents.zip trading-agents
   ```
   Windows PowerShell:
   ```powershell
   Compress-Archive -Path trading-agents -DestinationPath trading-agents.zip
   ```
3. Open the Skills section in Claude's settings and upload `trading-agents.zip`. Depending on the app version, it is located under **Settings > Capabilities** or **Customize > Skills**.
4. Make sure the skill is switched on.

### 2b. Install on Claude Code

Copy the skill folder into your personal skills directory (available in every project):

```bash
mkdir -p ~/.claude/skills
cp -r trading-agents ~/.claude/skills/
pip install -r requirements.txt
```

Windows PowerShell:
```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse trading-agents "$HOME\.claude\skills\"
pip install -r requirements.txt
```

To use it in a single project only, copy it to `.claude/skills/` inside that project instead. Using a virtual environment for the Python packages is recommended.

## Usage

Start a new chat and ask naturally. The skill is triggered by requests to analyse or rate a stock or crypto asset.

```text
Analyse NVDA
Run a deep analysis of BBCA.JK, I have 50 million rupiah to invest
Quick check on BTC
Compare BBRI, BMRI and BBNI
Analyse AAPL as of 2025-03-01
```

Indonesian prompts work the same way:

```text
analisis saham BBCA
BBRI layak beli?
analisis mendalam TLKM, saya punya modal 50 juta
```

Useful options you can mention in the prompt:

| You say | Effect |
|---|---|
| "quick" | Shorter reports, 1 debate round |
| "deep" or "thorough" | Longer reports, 3 debate rounds, 2 risk rounds |
| "only technical and fundamentals" | Runs a subset of analysts |
| "as of 2025-03-01" | Analyses a past date (backtest mode) |
| Your capital, average price or risk tolerance | Used by the Trader and Portfolio Manager for sizing and advice framing |

Ticker formats follow Yahoo Finance: `BBCA.JK` for Indonesia, `AAPL` for the US, `BTC-USD` for crypto, `0700.HK`, `7203.T` and so on for other markets. A bare four-letter code in an Indonesian context is treated as an IDX stock.

### Getting better results on claude.ai

- **Upload a daily price file (1 to 2 years)** together with your request. This enables full technical analysis. Exports from Investing.com, Yahoo Finance, TradingView or your broker all work. A benchmark file (for example IHSG) additionally enables beta and relative strength.
- **Or allow Yahoo Finance hosts** in your network settings if your plan offers that option. The script reports the blocked host name; `yfinance` typically needs `query1.finance.yahoo.com`, `query2.finance.yahoo.com` and `fc.yahoo.com`.
- **Keep the decision log.** Download `trading_agents_log.jsonl` after an analysis and upload it with your next request so the Portfolio Manager can review earlier calls.

### What you get

- A summary in the chat: final rating, confidence, horizon, entry, stop, targets, size, each desk's view, the strongest bull and bear points, and what would change the view.
- A full report `{TICKER}_{DATE}_trading-agents.md` with every agent's output, the data snapshot and the sources.
- An updated `trading_agents_log.jsonl`.

## Running the scripts on their own

The helper scripts also work from a terminal without Claude.

```bash
cd trading-agents/scripts

# Technical snapshot from any daily price CSV
python indicators.py --csv prices.csv --ticker BBCA.JK --date 2026-09-29 \
    --benchmark-csv ihsg.csv --md technical.md --out technical.json

# Fetch everything from Yahoo Finance (needs internet access to Yahoo)
python fetch_data.py BBCA.JK --date 2026-09-29 --outdir runs/BBCA

# Tick-valid stop, targets and position size
python risk_tools.py --ticker BBCA.JK --entry 9800 --atr 160 --stop-atr 2 \
    --targets-r 1.5,3 --equity 100000000 --risk-pct 1

# Decision memory
python decision_log.py add --ticker BBCA.JK --rating Overweight --price 9800 --thesis "..."
python decision_log.py review --ticker BBCA.JK --price 10150
```

Run any script with `--help` for all options.

## Repository structure

```text
trading-agents-lite/
├── README.md                 English documentation
├── README.id.md              Indonesian documentation
├── requirements.txt          Python packages for the helper scripts
├── assets/
│   ├── agent-flow.svg        Flow diagram (English)
│   ├── agent-flow.id.svg     Flow diagram (Indonesian)
│   └── diagram.png           Detailed flow diagram (generated by gitdiagram.com)
└── trading-agents/           The skill itself (this folder is what you install)
    ├── SKILL.md              Workflow and orchestration instructions
    ├── references/
    │   ├── agents.md         Role cards for all 12 agents
    │   ├── data-sources.md   Data paths, fallbacks, search recipes, market rules
    │   └── output-format.md  Chat summary and full report templates
    └── scripts/
        ├── common.py         Market detection, benchmarks, IDX ticks and lots
        ├── fetch_data.py     Yahoo Finance fetch with look-ahead filtering
        ├── indicators.py     Technical snapshot from any OHLCV file
        ├── risk_tools.py     Stops, targets, reward to risk, position size
        └── decision_log.py   Decision memory and outcome review
```

The skill instructions are written in English for precision. Claude answers in the language you use.

## Credits

The agent roles, the debate structure and the five-tier rating scale follow the design of TradingAgents by Tauric Research. If you use this work, please also credit the original:

```bibtex
@misc{xiao2025tradingagentsmultiagentsllmfinancial,
      title={TradingAgents: Multi-Agents LLM Financial Trading Framework},
      author={Yijia Xiao and Edward Sun and Di Luo and Wei Wang},
      year={2025},
      eprint={2412.20138},
      archivePrefix={arXiv},
      primaryClass={q-fin.TR},
      url={https://arxiv.org/abs/2412.20138},
}
```

## Disclaimer

This project is for research and education. Its output is generated by a language model from data that may be incomplete, delayed or wrong. It is not financial, investment or trading advice. Markets carry risk, including the loss of your capital. Do your own research and consider a licensed professional before making decisions.
