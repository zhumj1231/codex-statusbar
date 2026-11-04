# Usage and timing fix (local build)

Version: `1.0.0-rc3.usage-fix1`

## What changed

`token_usage_record.thread_token_usage` and `token_count.info.total_token_usage` can have different
cumulative baselines even when their per-response usage is identical. The parser now selects the
modern carrier once it is available and retains `token_count` only as a fallback for older logs.
It never subtracts one carrier's baseline from the other's total. Delayed, regressive snapshots
of the selected source are ignored and reported as `stale-usage:<carrier>`.

Modern responses are deduplicated by `response_id`, with the cumulative snapshot as a fallback.
Legacy responses use the cumulative snapshot as their identity. Two different responses with
identical input/output counts are no longer mistaken for one response.

The previous rate could combine updated item timing with old output counts, or include all tool
argument tokens while timing only a short reasoning item. A sample is now committed only at a
response's usage record. For modern logs, its span starts at the earliest timed Reasoning or
AgentMessage item since the previous response and ends at the usage record timestamp. This includes
argument-generation time between the last model item and the end of the response, while subsequent
tool execution and delayed `token_count` delivery do not extend that response's time.

While a turn is running, the display smooths completed-response rates with EMA alpha 0.35. At turn
completion, it displays `sum(matched output tokens) / sum(matched response seconds)`, not the final
EMA. A response without a usable starting time is counted in the token totals but excluded from
both sides of the speed calculation. The last measured value is held with `~`; before any valid
measurement, speed is `--`. A final average with untimed responses is also marked `~`.

Older records without a response timestamp retain item-window-union timing for text responses.
Legacy tool calls with no response completion timestamp are not measured: the later token-count
notification can arrive after tool execution, so its arrival time is not a safe substitute.

`ContextCompaction` is a known non-output item. File reinitialization clears source preference and
all parser state. Incremental reads no longer concatenate an unfinished tail twice, re-read an
unchanged partial line, or discard a partial string merely because it ends in a closing brace.

## Limits and diagnostics

This remains an **estimate from local response-level logs**, not a stream of server-side token
timestamps. Log flush delays and gaps within a response can affect it; hidden reasoning is included
in official output usage. An incomplete tail with no observed task start has unknown speed. Missing
timing is not filled with character counts or another response's duration.

Debug/probe snapshots expose `MeasuredOutputTokens`, `MeasuredGenerationMs`, and `UntimedResponses`.
`TurnOutputTokens` includes all observed responses from the chosen source, while the measured fields
describe only the sample coverage. Normal operation still writes no diagnostic log by default.

## Validation

- Self-tests cover distinct cumulative baselines, equal-sized consecutive responses, duplicate and
  stale records, late usage from another turn, item/usage arrival order, tool-only responses without
  timing, weighted final averages, compaction and split JSONL writes/reinitialization.
- The existing relative-clock fixture intentionally omits response timestamps; new tests use
  coherent Unix timestamps and response boundaries.
- A local replay of one completed 26-response turn independently summed 9,715 output tokens.
  Of these, 9,217 tokens had usable spans totaling 164,305 ms; 2 responses had no start time.
  The fixed completed-turn estimate was 56.0969 tok/s, matching the independent calculation.
  The old parser reported 246.7707 tok/s on the same records.
- The last modern cumulative total in that replay was 14,887,750; the legacy total was 9,457,287.
  The fixed parser retained the modern value instead of switching to the legacy value.

No private conversation text, actual thread identifiers or original logs are included in this repository.
