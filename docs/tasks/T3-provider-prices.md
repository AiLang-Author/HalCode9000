# T3: price fields for the cost meter

## Goal

- Add per-model prices to the provider files, so roadmap item 5 (the
  cost/token meter) has data.
- This task covers the schema and data only. The meter itself comes
  after step 0.

## Where

- Work only in `/home/bob/HalCode9000/providers/*.json`: anthropic,
  deepseek, gemini, groq, ollama, openai, openrouter, xai.
- Other trees get the change later through the step 0 sync. Don't edit
  them now.

## Do

1. Add these optional fields to each model entry, in USD per million
   tokens:
   - `price_in`;
   - `price_out`;
   - where the provider charges for cache: `price_cache_read` and
     `price_cache_write`.
   Add a provider-level `prices_checked` date (YYYY-MM-DD) and
   `prices_source` (the URL of the provider's own pricing page).
2. Fill prices only from each provider's official pricing page, current
   on the day you check.
   - If a model's price can't be confirmed, leave its fields out. Don't
     guess.
   - Leave all prices out for Ollama (local); it costs nothing per
     token.
   - For OpenRouter, take prices from its models API, not from the
     upstream providers.
3. Make sure `/refresh-models` (`CC_RunModelScraper` and its scraper
   script) keeps the price fields when it rewrites a provider file. If
   it would drop them, fix it so it merges.
4. Check that every file still parses as JSON. Make sure HalCode still
   loads the providers: run `--mcp` initialize and `tools/list` once
   with the trusted compiler's build in a scratch path.

## Rules

- Don't replace `Ailang-Self-Hosting-/ailang.x`. Don't commit or push.
- Don't put any API key into any file, and don't call paid model APIs.
  Reading public pricing pages and model-list endpoints is fine.

## Output

- Report:
  - a table of provider, model, the four prices, and the source URL;
  - the models left without prices, and why;
  - whether the scraper had to be changed;
  - the parse and load check results.
