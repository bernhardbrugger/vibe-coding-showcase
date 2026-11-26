# Case studies

Each one says what was built, how the agent worked, and where it was wrong. The mistakes are part of the point: the system is judged by how it catches them.

## 1. A private market-data hub (300+ tests)

**What.** A read-only toolbox for retail traders: connectors for central-bank, statistics, regulatory and prediction-market data, a local store that remembers when it first saw each value, descriptive checks on prices, an event study, a portfolio page, a command line, an MCP server and a local dashboard. No signals, no advice, no orders.

**How the agent worked.** It followed a roadmap in steps; each source needed a recorded real answer as a test fixture, a terms note and a concrete question it answers. Tests run on Linux and Windows in CI.

**Where it was wrong, and how it was caught**

- *Made-up probability.* A prediction-market contract with a bid of 1 cent and an ask of 93 cents was shown with a 47% probability (the midpoint). A live run before the commit showed the nonsense; quotes far apart are no longer averaged into a value.
- *Labels that meant nothing.* The first thresholds for "unusual" flagged pure noise in 0.3%, 1% and 23% of simulated series. They were re-calibrated by simulation to about 5%, the calibration is a test, and the daily brief now says how many flags chance alone would give.
- *CI red on a clean machine.* Tests had silently depended on the owner's own environment variables. A fixture now removes them, so tests run the same everywhere.
- *A layout that overlapped.* A new nine-column table was drawn over the next column; the owner's screenshot showed it. It was fixed and measured in a browser at three widths.
- *An invented example output.* A README example had been written before it was run. It was replaced by the real output before the commit, and "copy examples from a real run" became a rule.

## 2. A curated list with checks (awesome-ai-trading)

**What.** A list of tools, data and papers for AI-assisted trading research, with a written contribution policy, scripts that check links, repository facts and arXiv metadata, and a monthly automated link check.

**How the agent worked.** Every entry was checked for activity, licence and what it really does; papers were verified against their DOI metadata; a backtester comparison table states licences read from the repositories (one of them carries a clause that forbids selling products built on it).

**Where it was wrong, and how it was caught.** It first overwrote an existing pull-request template. A diff review caught it, and the next commit restored the original and added only what was new. A pull request from a directory operator was declined after reading the page it linked to: the footer disclosed affiliate links, which the contribution policy excludes.

## 3. A published MCP server (parlament-at-mcp)

**What.** An MCP server for the Austrian parliament's open data, published to npm and the registry, with CI, security scanning and a **live canary** that exercises the real interface on a schedule, so drift in the upstream API is noticed before users do.

**How the agent worked.** Twice a week a maintenance run checks CI, advisories, issues and the canary; dependency fixes go through the normal pipeline.

## 4. Small packages kept honest

- *A Bayesian A/B testing package* promised an interval type it did not implement; the promise was kept by writing it. A stopping rule on expected loss was added with a simulation that peeks after every batch, to check the claim that makes the rule useful.
- *A data-format skill* advertised XML it never supported, and silently read an unknown file type as CSV (producing an empty result). It now refuses with a message, the format table is verified cell by cell by tests, and the description says what is true.

## 5. Three red nights in a row (what-reddit-is-buying)

**What.** A scheduled GitHub Actions job that searches forum threads every weekday, has a language model summarise what people say they are buying, and publishes a daily board. It deliberately keeps the old board when the search comes back without forum text, rather than publishing a summary of nothing.

**How the agent worked.** Every daily run treats a red scheduled run as the first item of the day. It reads the failed log, decides whether the cause is in the code or outside it, and does not start a manual re-run just to see green, because each run spends paid search quota. The next scheduled run is the proof.

**What happened, night by night**

- *17 Nov, red.* The forum search returned 0 results for all three queries; the job stopped on purpose and kept the old board. Diagnosis: an outage outside the code, nothing to fix yet. Note for the next day: a second red night means look at the query and the engine.
- *18 Nov, cancelled.* A second failure, with a different cause, this time in the code. The search worked partly; then the call to one of the three model providers hung for eight minutes until the job's ten-minute limit killed it. That adapter was the only one without a request timeout, so neither the retry nor the fallback model ever ran. Fix: one shared 180-second timeout for every provider ([`8a1fa69`](https://github.com/bernhardbrugger/what-reddit-is-buying/commit/8a1fa69)), shown against a local server that accepts connections and never answers (the old code was still hanging after 20 s, the new one raised a timeout after 2.5 s) and pinned by a test that fails without the fix. The job limit was raised to fit the retries ([`eb33660`](https://github.com/bernhardbrugger/what-reddit-is-buying/commit/eb33660)).
- *19 Nov, red.* The forum engine was completely empty again, and the existing retry used the same engine, so it was empty too. Fix: an empty forum search is now repeated once on the regular search engine with the same query ([`da23712`](https://github.com/bernhardbrugger/what-reddit-is-buying/commit/da23712)), with a test that fails on the old code. It could not be proven locally, because the search key lives only in the repository's secrets.
- *20 Nov, green.* The scheduled run succeeded and the board was updated ([`18be376`](https://github.com/bernhardbrugger/what-reddit-is-buying/commit/18be376)). The forum engine was still empty that night; the fallback carried the run.

**Where it was wrong, and how it was caught.** The README promised a board before the US market opens, because the cron line says 13:30 UTC. Checked against the run history, the scheduled runs had actually started between 20:35 and 22:58 UTC, after the close. The claim was withdrawn ([`4821586`](https://github.com/bernhardbrugger/what-reddit-is-buying/commit/4821586)), and "check freshness claims against the run history, never against the cron line" became a written lesson. The week was also a reminder that the same symptom can have a different cause each night: the second and third failures were diagnosed from their own logs, not assumed to be the first one again.

## 6. Names that rot while the tests stay green (agora, bayesian-ab-testing)

**What.** Two Python projects with green CI: agora, a multi-agent debate tool with three language-model providers, and a Bayesian A/B testing package. Both passed every test, and both handed users a command or a default that did not work.

**How the agent worked.** Repositories that nobody has touched for a while get an end-to-end check in the work runs: a fresh virtual environment, a git install as a stranger would do it, every README example run and compared with the printed output, then every message the program shows a user read for the names it points at.

**What it found**

- *A hint that installed someone else's package.* Without matplotlib, the A/B package told users to run `pip install bayesian-ab-testing[plot]`. That name on PyPI belongs to an unrelated project by other authors, so following the hint would have installed foreign code. The hint now names matplotlib ([`e3a4973`](https://github.com/bernhardbrugger/bayesian-ab-testing/commit/e3a4973)); a test hides matplotlib and fails on the old text.
- *The same mistake, one repository over.* A search for the pattern across all Python repositories found agora telling users to `pip install "agora-debate[mcp]"`, a name that is not registered at all: the command fails today, and anyone could register the name and receive those installs. The hints now name the real dependency ([`b1231c6`](https://github.com/bernhardbrugger/agora/commit/b1231c6)).
- *A default that had been dead for five months.* agora's default model alias, written in February, pointed to a model the provider's deprecation page lists as retired in June 2026, so every real run without `--model` would have ended in a "not found" error. The other provider's default had been shut down two weeks before that. The provider tests use fake SDKs, so they could never notice. The aliases now point to current models read from the providers' own model and deprecation pages ([`bf744d1`](https://github.com/bernhardbrugger/agora/commit/bf744d1), [`2cf9fec`](https://github.com/bernhardbrugger/agora/commit/2cf9fec)), and the first test for that provider pins the mapping. They were not called live: no key is available to the agent, and the commits say so.

**Where it was wrong, and how it was caught.** Both install hints came from the agent's own earlier commits: plausible names for packages that were never published, and tests that checked a hint appeared, not that it worked. Two written lessons came out of it: an install hint names only a package that really exists under our name (checked against the PyPI JSON API), and alias tables in language-model tools are checked against the providers' deprecation pages during maintenance, because fake-SDK tests stay green whatever the real API says.

## What these have in common

The agent is fast, and also confidently wrong now and then. What made the difference was a loop that does not trust it: run the thing, read the real output, make the test able to fail, check CI, and let a human decide what is theirs.
