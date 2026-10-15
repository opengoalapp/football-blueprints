I'd like to build a small browser app for chatting with football data, as a demonstration of this
pattern: **give an LLM a few fixed data functions and let it answer by writing
short Python programs that call and combine them, run safely in a sandbox.**

# Background

The usual ways of putting an LLM in front of data each have a weakness.

- **Text-to-SQL** asks the model to cram the whole question into one query. That's fine for simple
  filters. It's brittle for anything that needs several steps, such as "find X, then look Y up for
  X", "do this for every team, then compare" or "rank within a group". It also exposes every column
  and its quirks, so the model has to be taught the data's subtleties in the prompt, and it will
  still get some of them wrong. In my experience the errors are mostly *semantic*: valid SQL that
  answers a slightly different question.
- **JSON tool calling** is safer, but every call is a round trip through the model. A question
  that needs five lookups and some arithmetic costs five or six LLM calls, and the model has to
  carry intermediate results in its context.
- **Letting the model write and run real code** is the most expressive option, but running
  untrusted generated code normally means containers or a remote sandbox.

"Code mode" combines the good parts. You expose a small set of well-designed functions, and the
model writes a short program that calls them. It can loop, join, rank and chain results, all in one
round. It's also firmly constrained: the functions are the *only* way to the data, so anything
fiddly about the data is handled inside them once, and the prompt never has to explain it.

What makes this practical in a browser is **Monty**, Pydantic's minimal Python interpreter written
in Rust and compiled to WebAssembly. It starts in milliseconds and has no filesystem, network or
imports of its own. The sandboxed code can only reach the outside world through the functions you
hand it.

# The idea, independent of any dataset

Designing the functions:
- Keep them few, and make each return a list of plain dicts with one clear row grain (for example,
  "one row per player").
- Name them as verbs (`get_runs`, not `runs`). Models happily call their own variables `runs` and
  then can't call the function any more.
- Make the arguments forgiving: partial, case-insensitive names. When an argument is unknown or
  ambiguous, raise an error that lists the valid options, so the model can correct itself in the
  next round.

The conversation loop:
- The model replies with one fenced python block. Run it, then send back what it printed and the
  value of its last expression, or the traceback if it failed.
- Repeat until it replies without a code block; that reply is the answer. Cap it at a handful of
  rounds (five works well), then ask it to answer with what it has.
- Keep user and assistant messages strictly alternating. Some models' chat templates reject two
  user messages in a row. Add the round-limit note to the last execution result rather than
  sending a new message, and if a request fails, drop the unanswered question from the history.

The system prompt should stay short:
- the loop above;
- each function's signature and fields, in a line or two each;
- the valid names for any filters;
- what the sandbox can't do (no pandas, numpy, `match` or class inheritance).

It also needs a few rules that models otherwise get wrong:
- top-level `await` works (otherwise they wrap everything in `asyncio.run(main())`);
- only printed output and the last expression come back to them;
- the answer must only use numbers they have actually seen, and should mention ties. Without this,
  a model that never printed its rows will happily invent a table.

# Things about Monty that may trip you up

Monty is `@pydantic/monty` on npm, version 1.x. Older 0.x examples on the web use a different API.
- **Bundling:** use the browser build `@pydantic/monty/wasm`. It runs in its own Web Worker and
  can't be loaded from a CDN, so use Vite with `optimizeDeps.exclude: ['@pydantic/monty']` and
  `worker.format: 'es'`.
- **Running code:** `Monty.create()`, then `pool.checkout({limits})` gives a session, then
  `session.feedRun(code, {externalLookup, printCallback: new CollectString()})`.
  - `feedRun` returns the value of the last expression.
  - Python dicts come back to JavaScript as `Map`s.
  - Keyword arguments reach a JavaScript function as a trailing plain object.
- **Async functions:** if the data functions are async, the Python must `await` them, and
  `asyncio.gather` runs several at once.
- **Suspension budget:** every `await` counts against `maxSuspensions`, which defaults to 1000 for
  the whole session. Raise it a lot.
- **Errors and sessions:**
  - Keep one session alive so variables persist between rounds.
  - An error thrown by a JavaScript function becomes a Python exception the model can read.
  - Syntax and runtime errors offer `.display('traceback')`.
  - If the sandbox itself crashes, start a new session.
- **Overwritten functions:** pass the functions as `inputs` as well as `externalLookup`, and give
  the JavaScript functions real names (Monty resolves `inputs` by function name). That re-binds
  them before every run, so a model that once overwrote one doesn't lose it for the rest of the
  session.
- **`next()`:** generator expressions evaluate to lists in Monty, so `next(x for x in ...)` fails.
  Models write it constantly whatever the prompt says, so run a few lines of Python at the start of
  each session that redefine `next()` to accept a list too.

# The demo

A single-page app that runs entirely in the browser apart from the LLM call. Use Vite and vanilla
TypeScript, no framework. Keep it a minimal working demo.

**Data.** SkillCorner's open data (the `SkillCorner/opendata` repository on GitHub) includes
season-level aggregates for the 2024/25 Australian A-League in `data/aggregates/`: three small CSVs
for physical, passing and off-ball-run metrics. Read them straight from raw.githubusercontent.com
into DuckDB-WASM when the page loads; there's no download step.
- The CSVs have one row per player *per position group*. Keep each player's most-played position,
  so every function returns exactly one row per player.
- Two different players share a name, so include `player_id` and join on that.
- Open DuckDB-WASM with `castBigIntToDouble` and `castDecimalToDouble` set. An integer `SUM` comes
  back as a 128-bit integer, and Monty needs plain numbers.

**The functions.** All three are async and take optional `team` and `position` filters. Each
returns `player_id, player, team, position, matches` plus a few metrics with readable names:
- `get_physical()`: top speed (PSV-99), distance, metres per minute, high-speed running distance,
  sprint distance and sprint count, all per match.
- `get_passing()`: passes, completion %, and completed line-breaking and dangerous passes, all per
  30 minutes of team possession.
- `get_runs()`: off-ball runs, runs in behind, dangerous runs, and runs followed by a shot within
  10 seconds, all per 30 minutes of team possession.

Filters accept partial names like "Auckland" or "centre forward". Watch the spelling: the data says
"Center". Give the system prompt the list of team and position names.

**Choosing the LLM.** People will have their own setup, so let them pick a provider in the header.
Remember their choice, and the model chosen for each provider, in the browser.
- **OpenRouter**, which allows calls straight from the browser.
  - Offer "Sign in with OpenRouter" using its OAuth PKCE flow: redirect to OpenRouter's auth page
    with a SHA-256 code challenge and this page as the callback, then exchange the returned code
    for the user's own key. Also accept a pasted key.
  - The model list is public, so it loads before sign-in. Show free models first (both prices are
    "0") and leave out models that output anything other than text.
  - Default to OpenRouter's free router (`openrouter/free`). Individual free models are often
    rate-limited by their upstream provider, and the router picks one with capacity.
  - Turn 401, 402 and 429 errors into plain messages. For 429, show OpenRouter's own reason, which
    usually blames the upstream provider rather than the user's quota.
- **A local OpenAI-compatible server**, such as llama.cpp, Ollama or LM Studio.
  - Reach it through the dev server's proxy, so the local server doesn't need CORS.
  - Take its address from an `LLM_URL` environment variable, defaulting to llama.cpp's usual
    `http://127.0.0.1:8080`.
  - List the models it reports.
- **For both:** stream the replies, and use a generous timeout, because local models can be slow.
  Reasoning models send their thinking separately: llama.cpp calls it `reasoning_content`,
  OpenRouter calls it `reasoning`, and OpenRouter also sends keep-alive comment lines.

**The interface.** Make it look considered, not like browser defaults.
- **Header:** the provider and model pickers, sign-in, and a status line. Chat on the left, a
  "trace" panel on the right, light and dark mode, and a "New chat" button that clears the
  history and the sandbox.
- **Answers:** render them as Markdown (sanitised), so tables and bold text look right, with the
  raw result rows in a collapsible section underneath.
- **The trace is where the idea becomes visible:**
  - stream each round into it as the model writes, reasoning included, so you watch the program
    being written;
  - then show the finished round as a card: the code, a chip for each data call (with its
    arguments, row count and time), and the output or error.

**Starter pills.** Above the input, three pills. Clicking one fills the input rather than sending
it. Each has one clear reading and shows a different kind of composition:
1. "Who is the fastest player in each team, by top speed?" (group and pick, in Python)
2. "Who completes the most line-breaking passes per 30 minutes (at least 5 matches), and where
   does their top speed rank within their own team?" (one result drives the next call)
3. "Which centre forwards with at least 5 matches are in the top 10 for both runs in behind and
   top speed?" (two datasets joined, then intersected)

**When you're done,** leave a short note on how to run it (`npm run dev`, and `LLM_URL` for a local
server). It works when each pill gets a sensible answer from OpenRouter's free router in a round
or two, with the trace showing the code, the calls and the result.
