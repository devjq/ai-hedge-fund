# CLAUDE.md

Notes for running this project that aren't obvious from the README.

## Running without Poetry

The README assumes Poetry. If Poetry isn't installed, a plain venv works:

```bash
python3 -m venv .venv
.venv/bin/pip install \
  "langchain>=0.3.7,<0.4" "langchain-anthropic==0.3.5" "langchain-groq==0.2.3" \
  "langchain-openai>=0.3.5,<0.4" "langchain-deepseek>=0.1.2,<0.2" \
  "langchain-ollama==0.3.6" "langgraph==0.2.56" "langchain-core>=0.3.73,<0.4.0" \
  "langchain-google-genai>=2.0.11" "langchain-gigachat<0.5" "langchain-xai<1.0" \
  "python-dotenv==1.0.0" pytest pydantic requests pandas matplotlib tabulate \
  colorama questionary rich scipy
```

`langchain-gigachat>=0.5` and `langchain-xai>=1.0` pull in `langchain-core>=1.0`, which breaks every other adapter. Pin them below those versions.

When running without `poetry install`, the `src` package isn't on `sys.path`, so prefix commands with `PYTHONPATH=.`:

```bash
PYTHONPATH=. .venv/bin/python src/main.py --tickers AAPL ...
```

## Running unit tests (no API keys needed)

```bash
.venv/bin/python -m pytest tests/ -v
```

Both test files (`test_cache.py`, `test_api_rate_limiting.py`) use mocks — no network, no keys.

## CLI gotchas

- Flag is `--tickers` (plural), **not** `--ticker` as the README shows.
- `--analysts` takes snake_case names (`warren_buffett`, `cathie_wood`, `michael_burry`, ...) matching files in `src/agents/`.
- Without `--model`, the CLI drops into an interactive `questionary` picker — supply `--model gpt-4.1` (or similar) for non-interactive runs.

## Data source constraint (important)

The project is hardwired to `api.financialdatasets.ai` (same in `src/tools/api.py` and `v2/data/client.py`). There is no built-in alternative data source.

Financial Datasets' **free tier** (whether a key is set or not) only returns data from roughly the current date onwards — historical queries return HTTP 403 with `"Your plan includes data from <date> onwards. Upgrade to Pro..."`. Practical effects:

- Price data works for the last ~3 months through today.
- Financial metrics return ~5 recent periods.
- Backtesting over multi-year history is not possible without Pro.
- If you pass a date range the free tier rejects, `get_prices` etc. silently return empty lists → Risk Manager errors with `"Missing price data for risk calculation"` → Portfolio Manager outputs HOLD with "No valid trade available".

**First debugging step when agents produce HOLD / "insufficient data"**: probe the price fetch directly:

```python
from dotenv import load_dotenv; load_dotenv()
from src.tools.api import get_prices
print(len(get_prices("NVDA", "<start>", "<end>")))
```

If it returns 0, your date range is outside the free-tier window.

## Known-good commands

Single analyst, current window:

```bash
PYTHONPATH=. .venv/bin/python src/main.py \
  --tickers NVDA \
  --analysts cathie_wood \
  --model gpt-4.1 \
  --start-date 2026-01-02 --end-date 2026-04-22 \
  --show-reasoning
```

Multiple analysts + tickers:

```bash
PYTHONPATH=. .venv/bin/python src/main.py \
  --tickers NVDA,AAPL,TSLA \
  --analysts warren_buffett,michael_burry,cathie_wood \
  --model gpt-4.1 \
  --start-date 2026-01-02 --end-date 2026-04-22
```

## .env

Only `OPENAI_API_KEY` is strictly required to test the pipeline (or any other LLM provider key — see `.env.example`). `FINANCIAL_DATASETS_API_KEY` is optional on the free tier: the data returned is the same whether the key is set or not, because the 2025-04-23 cutoff is tier-based, not key-based.

## v2/ folder

`v2/` is an in-progress quant rebuild (see `v2/README.md`). It uses the **same** Financial Datasets API as the sole data provider, so it inherits the same free-tier constraint. It is not yet wired up to a CLI — there's no end-to-end entry point, only the data client and building blocks.
