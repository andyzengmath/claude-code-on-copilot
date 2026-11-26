# Speed comparison: 2026-11-26

Does Claude Code feel slow through this proxy? This report compares the speed of
Claude Opus 5.5 on four paths: the proxy's API, Claude Code through the proxy,
GitHub Copilot CLI, and Claude Code on Anthropic's own API.

In short, neither the proxy nor Copilot is the cause. The proxy adds no
measurable latency, and Copilot streams tokens at the same rate to Claude Code
through the proxy as to Copilot CLI. Claude Code's turns take longer because it
runs Opus at a higher effort level, so each call generates more tokens, and
because very large contexts make the slowest calls slower.

**Scope:** Claude Opus 5.5 (`claude-opus-5.5` on Copilot), the proxy at
`6b147c4`, Claude Code 2.1.327–2.1.332 and Copilot CLI 1.0.114. The real
sessions are one user's work on a Windows 365 Cloud PC, with three Claude Code
sessions open at the same time.

**Method:** two kinds of evidence. Timings from real sessions cover 8,104
Claude Code calls through the proxy (17–26 November) and 37,307 Copilot CLI
calls (12–26 November). A small benchmark sent one identical prompt through
each path. [Method](#method) gives the details.

## Benchmark: the same prompt on every path

The prompt asked for the integers from 1 to 400, which produced a reply of 801
output tokens. Each path ran twice, interleaved; the table lists both runs.

| Path | First token | Call time | Generation speed |
|---|---|---|---|
| 1. Proxy API | 0.55 s, 0.44 s | 5.65 s, 5.45 s | 157, 160 tokens/s |
| 2. Claude Code through the proxy | 1.80 s, 0.78 s | 6.78 s, 5.86 s | 160, 160 tokens/s |
| 3. Copilot CLI | 1.06 s, 0.85 s | 6.19 s, 6.10 s | 162, 162 tokens/s |
| 4. Claude Code on Anthropic's API | Not measured | Not measured | Not measured |

- **First token** and **call time** exclude each CLI's startup. With startup,
  Claude Code's runs took 14–16 s. In a separate probe run, Copilot CLI took
  about 28 s from launch to its first model call. Both costs come once per
  session in interactive use.
- **Path 4** failed before reaching the model: Anthropic rejected the stored
  `/login` key with "400 Not a valid API key for this workspace".

## Real sessions

| Claude Opus 5.5 | Claude Code through the proxy | Copilot CLI |
|---|---|---|
| Calls | 8,104 | 37,307 |
| Mean context per call | 216k tokens | 361k tokens |
| Mean output per call | 2,498 tokens | 1,408 tokens |
| Mean time per call | 22.6 s | 13.2 s |
| Output tokens per second, all calls | 110.5 | 106.8 |
| Output tokens per second, median for outputs of 2,000+ tokens | 115.6 | 115.1 |

Speed in real sessions is output tokens divided by the whole call, so it
includes the wait for the first token.

## Findings

### 1. The proxy adds no measurable latency

Claude Haiku 5.5 answered the benchmark prompt three times through the proxy and
three times directly from Copilot, alternating, with the proxy's own headers.

| Pair | Through the proxy: first token, call, speed | Directly from Copilot: first token, call, speed |
|---|---|---|
| 1 | 0.68 s, 2.89 s, 363 tokens/s | 0.76 s, 2.98 s, 361 tokens/s |
| 2 | 0.46 s, 2.65 s, 367 tokens/s | 0.61 s, 2.82 s, 362 tokens/s |
| 3 | 0.55 s, 2.76 s, 363 tokens/s | 0.54 s, 2.75 s, 363 tokens/s |

### 2. Tokens arrive at the same rate on every Copilot path

The three Copilot paths generated 157–162 tokens/s in the benchmark. In real
sessions, long outputs reached a median of 115.6 tokens/s through the proxy and
115.1 in Copilot CLI. Through the proxy, the daily median stayed between 94 and
107 tokens/s from 17 to 26 November.

### 3. Effort level makes Claude Code's calls longer

Claude Code ran Opus mostly at `xhigh` or `max` effort. A higher level thinks
longer and writes more tokens per call; the token rate does not change. Claude
Code calls through the proxy, 19–26 November:

| Effort | Calls | Mean time per call | Median | 90th percentile | Mean output | Output tokens/s |
|---|---|---|---|---|---|---|
| `medium` | 129 | 12.1 s | 4.1 s | 33.0 s | 1,295 | 107.4 |
| `xhigh` | 1,633 | 16.4 s | 5.1 s | 33.7 s | 1,865 | 114.1 |
| `max` | 5,898 | 22.5 s | 5.6 s | 52.0 s | 2,530 | 112.7 |

All 15 Copilot CLI sessions that started on Opus 5.5 in the period started at
`medium`; six more recorded no model and started at `max`. Lower effort in
Copilot CLI accounts for most of the difference in output and time per call
between the two tools.

### 4. Large contexts make the slowest calls

Call time by context size (prompt plus cached tokens), median and 90th percentile:

| Context | Claude Code through the proxy | Copilot CLI | Copilot CLI first token |
|---|---|---|---|
| Under 50k | 3.4 s, 9.4 s (352 calls) | 3.2 s, 7.2 s (648) | 1.8 s, 2.9 s |
| 50k–200k | 4.5 s, 47.2 s (3,847) | 4.4 s, 22.4 s (10,018) | 2.5 s, 3.6 s |
| 200k–500k | 7.6 s, 52.1 s (3,680) | 7.0 s, 33.1 s (16,630) | 2.9 s, 4.1 s |
| 500k and over | 11.2 s, 126.7 s (225) | 7.3 s, 26.5 s (10,011) | 3.2 s, 5.4 s |

Below 500k tokens, typical calls take about the same time in both tools. The
slowest tenth of Claude Code's calls are much slower, because those calls also
think longest. Prompt caching served 97–100% of the context for calls of 50k
tokens and more in both tools.

### 5. Running several sessions at once does not slow requests

Opus calls through the proxy, 17–26 November, grouped by how many other calls
were in flight at the same time:

| Other calls in flight | Calls | Median output tokens/s | Output tokens/s, outputs of 1,000+ tokens |
|---|---|---|---|
| 0 | 856 | 101.2 | 111.3 |
| 1 | 974 | 103.3 | 117.7 |
| 2 | 763 | 103.2 | 116.8 |
| 3 | 690 | 102.4 | 116.1 |
| 4 or more | 4,821 | 105.5 | 114.9 |

### 6. Claude Code on Anthropic's API

The live comparison was not possible, because the stored key was rejected. Two
references remain:

- **The same user's Claude Code on Anthropic's API in May 2026**, with older models:

  | Model | Calls | Median time per call | Median output tokens/s | Output tokens/s, outputs of 1,000+ tokens |
  |---|---|---|---|---|
  | Opus 4.7 | 17,970 | 10.3 s | 51.0 | 67.9 |
  | Opus 4.6 | 11,747 | 6.0 s | 40.6 | 46.2 |
  | Sonnet 4.6 | 1,681 | 8.2 s | 41.1 | 52.3 |
  | Haiku 4.5 | 1,222 | 4.3 s | 33.2 | 83.9 |

- **Published figures for Opus 5.5.** [Artificial Analysis](https://artificialanalysis.ai/models/claude-opus-5-5/providers)
  measured about 97 output tokens/s on Anthropic's API in October 2026, about
  the same as on Amazon Bedrock, Google and Azure. It counts reasoning tokens
  over the full request at `max` effort, which matches how real sessions are
  measured here. [LLM Verifications](https://www.llmverifications.com/models/claude-opus-5-5)
  measured 129 tokens/s and 2.3 s to the first token, including thinking, in a
  27 October 2026 run.

Copilot's Opus 5.5 is at least as fast: 107–116 tokens/s in real sessions.

## Recommendations

1. Use `/effort medium` or `high` in Claude Code for routine work, and keep
   `xhigh` and `max` for hard problems. In this data, `medium` calls took about
   half as long as `max` calls.
2. Keep contexts small with `/compact` or a fresh session. At 500k tokens and
   more, the slowest tenth of Claude Code's calls took over two minutes.
3. The proxy needs no change for speed.
4. To measure path 4, sign in again with `claude /login` while
   `ANTHROPIC_BASE_URL` and `ANTHROPIC_AUTH_TOKEN` are unset, then rerun the
   benchmark.

## Method

### Benchmark

- **Prompt:** "Write the integers from 1 to 400 separated by single spaces.
  Output only the numbers, nothing else."
- **Proxy API:** a streamed `POST /v1/messages` to the proxy with
  `max_tokens: 4000` and no thinking. First token is the first `text_delta`;
  generation speed is output tokens divided by the time from the first token to
  the end.
- **Claude Code:** `claude -p "<prompt>" --model opus --effort medium
  --output-format stream-json --verbose --include-partial-messages
  --strict-mcp-config`. Times start at the `system`/`init` event, which excludes
  startup. First token is the first `content_block_delta`, call time is the
  result's `duration_api_ms`, and generation speed runs from the first to the
  last delta.
- **Copilot CLI:** `copilot -p "<prompt>" --model claude-opus-5.5
  --reasoning-effort medium --output-format json`. Times run from
  `model.call_start` to the first and last `assistant.message_delta` and to
  `model.call_finished`.
- **Claude Code on Anthropic's API:** the Claude Code command with
  `ANTHROPIC_BASE_URL` and `ANTHROPIC_AUTH_TOKEN` unset.
- **Proxy overhead:** Claude Haiku 5.5 through the proxy and directly at
  `https://api.githubcopilot.com/v1/messages`, alternating, three pairs.

### Claude Code sessions

- Transcripts in `~/.claude/projects/**/*.jsonl`. A call starts at an
  `api-request` entry; its response is the following assistant entries with the
  same `message.id`, and the call ends at the last of them. Tokens come from
  `message.usage`; context is input plus cache-read plus cache-creation tokens.
- Calls to Anthropic's API record a `requestId`; calls through the proxy do not.
  No November entry had one.
- Effort comes from each entry's `effort` field. Calls in flight at the same
  time are calls, from any session, whose start-to-end intervals overlap.

### Copilot CLI sessions

- `session.usage_record` events in `~/.copilot/session-state/*/events.jsonl`.
  They give each call's `timeToFirstTokenMs`, `duration`, `inputTokens`,
  `cacheReadTokens` and `outputTokens`. A session's effort comes from its
  `session.start` event.

### May 2026 sessions

- A May 2026 backup of Claude Code transcripts; every entry has a `requestId`.
  These transcripts have no `api-request` entries, so a call starts at the
  preceding user entry, which adds the client's own time.

## Limitations

- Claude Code transcripts record when each content block finished, not when the
  first token arrived. So time to first token through the proxy in real
  sessions is unknown. The proxy adds no measurable latency and calls the same
  Copilot endpoint, so it is expected to be close to Copilot CLI's.
- The benchmark ran two short calls per path at `medium` effort.
- Real sessions in the two tools did different work, with different contexts
  and effort levels.
- The May sessions used older models, and their call times include client time.
- Published figures use their own workloads and settings.
- Copilot CLI's session summary store gave 13.5 s per call and 79.8 tokens/s
  over 13,276 Opus calls in the same period. It covers fewer calls and does not
  say how it measures duration, so this report uses the per-call records.
- Copilot CLI compresses request bodies (median 1.44 MB to 0.67 MB); the proxy
  does not. The effect on upload time was not measured.
- Copilot CLI charged 15 premium requests per Opus 5.5 prompt. The benchmark
  used 45 there (two runs and a probe), plus two Opus calls to the proxy's API,
  two Claude Code runs through the proxy, and six Haiku calls.
