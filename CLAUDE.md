# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Python 3.10 with dependencies managed by Pipenv. Run everything from the repo root: modules import each other by top-level names (`from agents...`, `from tools...`), so they only resolve with the root on `PYTHONPATH`.

```bash
pipenv install                                # install dependencies
pipenv run python app.py                      # Flask dev server at http://localhost:5000 (debug on)
pipenv run python -m third_parties.linkedin   # run a single integration by itself (each module has a __main__)
pipenv run python -m third_parties.twitter
pipenv run black . && pipenv run isort .      # format (black/isort/pylint are in [packages])
pipenv run pylint agents chains third_parties tools *.py
```

The README says to run `pipenv run pytest .`, but the repo has no tests and `pytest` isn't in the Pipfile.

## Architecture

The app takes a person's name and returns a summary, two facts, three topics of interest, two ice breakers and their LinkedIn photo. Everything happens in one synchronous request.

1. `app.py`: `POST /process` (form field `name`) calls `ice_breaker.ice_break_with(name)` and returns JSON built from each Pydantic result's `to_dict()`. `templates/index.html` is the whole frontend: a form plus inline JS that renders that JSON.
2. `agents/`: two LangChain ReAct agents (`hub.pull("hwchase17/react")`, `gpt-4o-mini`). Both use `tools/tools.get_profile_url_tavily` (Tavily search) to turn the name into a LinkedIn URL and a Twitter/X username.
3. `third_parties/`: `linkedin.scrape_linkedin_profile` calls the Scrapin.io enrichment API and drops empty fields. With `mock=True` it reads a fixed Gist instead. `twitter.py` has a real Tweepy scraper and `scrape_user_tweets_mock`, which reads a fixed Gist of Eden Marco's tweets.
4. `chains/custom_chains.py`: three LCEL chains (`prompt | llm | PydanticOutputParser`), all on `gpt-3.5-turbo`, all fed the same `{information, twitter_posts}` input. Each prompt gets `format_instructions` from the matching parser in `output_parsers.py`, so changing an output field means editing the Pydantic model there, its `to_dict()`, and the JS in `index.html`.

Gotchas:
- `ice_break_with` always uses `scrape_user_tweets_mock`. The Twitter agent runs, but its username result is never used to fetch real tweets.
- Importing `third_parties/twitter.py` builds a `tweepy.Client` from `os.environ[...]`, so the app crashes at startup if the `TWITTER_*` variables are missing, even though tweets are mocked.
- Each request makes several paid calls: OpenAI, Tavily, Scrapin.io and the LangChain Hub prompt pull.

## Environment

`.env` is loaded with `python-dotenv`. The code reads `OPENAI_API_KEY`, `SCRAPIN_API_KEY`, `TAVILY_API_KEY`, `TWITTER_BEARER_TOKEN`, `TWITTER_API_KEY`, `TWITTER_API_KEY_SECRET`, `TWITTER_ACCESS_TOKEN` and `TWITTER_ACCESS_TOKEN_SECRET`. The README's variable list and `.env.example` don't match the code: the README uses different Twitter names, and `.env.example` lists `PROXYCURL_API_KEY` instead of `SCRAPIN_API_KEY`. Trust the code.

If `LANGCHAIN_TRACING_V2=true`, a valid `LANGCHAIN_API_KEY` is required or the app errors. Leave those variables out to disable LangSmith tracing.

## CI

`.github/workflows/claude-code-review.yml` runs Claude code review on PRs. `.github/workflows/claude.yml` responds to `@claude` mentions in issues and PRs. There's no lint or test CI.
