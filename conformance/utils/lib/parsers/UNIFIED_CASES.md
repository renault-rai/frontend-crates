# Unified Parser Cases (reasoning + content + tool calls, one ordered stream)

Reference taxonomy for the **unified** conformance surface: one parser owns the whole assistant-output grammar and emits ONE ordered event stream. Sibling stage docs: `REASONING_CASES.md` (reasoning only), `TOOLCALLING_CASES.md` / `TOOLCALLING_STREAMING_V1_CASES.md` (tool calls only). This surface is what those two cannot express — the ORDER between reasoning and tool calls, and reasoning that occurs *between* or *after* tool calls.

The golden corpus is authored by `conformance/utils/src/gen_unified_golden.py` (one scenario spec -> `conformance/unified/golden_spec/<family>.yaml` in the gitignored build tree); the committed canonical files under `conformance/fixtures-unified-v2/families/` are derived from it.

## Single-family case review

Always evaluate single-family tests against every registered family when adding, changing, or reviewing them, following the [case applicability policy](../../../README.md#case-applicability). If another family can express the behavior, add family-native inputs and authored expected outputs for every applicable family and use a shared section. Otherwise, explain the grammar or request-mode limitation and place the case in a named family-specific test section. Preserve published IDs, capture history, and older unmeasured results when changing section placement.

## The oracle: GOLDEN is authored, not captured

## Capture version policy

Follow [the Unified storage contract](../../../README.md#unified-storage-contract-plain-versioned-yaml-only): plain semantic-version YAML only, with unchanged family output carried forward. Source SHA is optional first-capture origin metadata, never a capture name. This migration does not change the older stream or batch-on-stream storage. Changing an existing Unified input requires rerunning and updating every prior affected version.

The shared-family coverage for `7-9`, `7-11`, `7-12`, and `7-13` is first measured in peer families at Dynamo v2 `0.7.18`. Earlier peer-family release cells remain unmeasured; they were not backfilled when coverage expanded. GLM was captured on the released Dynamo v2 `0.7.8` source under PR #271, and its existing history is preserved.

The truth column (`golden:`) is what a **correct** UnifiedParser MUST emit, reasoned from the invariants and policies below — NOT captured from vLLM, Dynamo, or any implementation. Both engines are measured against it and both can diverge (vLLM has documented spec violations: truncated-tool hard-error, streamed-arg truncation, trailing-text suppression). Never regenerate `golden:` from an engine; it is versioned like code.

## Event schema

One ordered list per case:

```yaml
golden:
  - {kind: reasoning, text: "..."}     # private chain-of-thought
  - {kind: tool_call, name: "...", arguments: {...}}   # final typed args object
  - {kind: text, text: "..."}          # user-visible content
```

Comparison is ORDER-SENSITIVE on the ASSEMBLED list. Streaming delta granularity may differ across engines; the assembled event list is the invariant (same principle as the tool-call chunk sweep).

## Invariants (every correct implementation must satisfy)

- **I1 Faithful segmentation** — every model byte is exactly one of reasoning / visible text / tool-call structure / control marker. Markers are consumed; nothing else dropped or duplicated.
- **I2 Order preservation** — events in the order the model emitted the underlying content (reason -> call -> reason -> text stays in that order).
- **I3 No marker leakage** — control markers never appear inside a text or reasoning payload.
- **I4 Per-stream isolation** — for n>1, each choice's events depend only on that choice's bytes: `demux(parse(interleave(s0,s1)))[i] == parse(s_i)`.
- **I5 Chunk-invariance** — assembled list identical for any chunk splitting.
- **I6 Stream/batch parity** — whole-output parse assembles to the same list as streamed.
- **I7 Argument fidelity** — arguments are the model's actual args, typed per schema; no fabricated/dropped/reordered keys; a marker-looking substring INSIDE a JSON string value is data, preserved exactly.
- **I8 Coalescing** — adjacent same-kind events merge.

## Governing principle: best-effort error recovery

The parser recovers everything it can and NEVER drops valid text, leaks markup, or hard-errors on malformed/truncated input. Documented contract: `conformance/README.md` (v2 "preserves surrounding/inter-call prose... recovers bare calls v1 drops"; "dropping text, leaking markup, corrupting args" is a regression to FIX, not paper over) and `TOOLCALLING_CASES.md` TOOLCALLING.batch.5.e / TOOLCALLING.batch.5.g (drop only the unrecoverable partial call while earlier output stays recoverable; strip orphan close markers; do not leak). This principle resolves the policy calls below.

## Policy decisions

- **P1 Trailing text after the last tool call** -> emit as `text`. RESOLVED by best-effort recovery: trailing prose is arbitrary visible content and must be preserved (dropping it is a regression). vLLM's kimi config suppresses it -> vLLM red (LOSS).
- **P2 Truncated tool call at EOF** -> DROP the unrecoverable partial call, emit preceding reasoning/text cleanly, no error, no leaked markup. RESOLVED by best-effort recovery (TOOLCALLING.batch.5.e). Dynamo drops -> correct. The two vLLM implementations fail differently, both confirmed by live 0.25.1 capture: the native Rust `Gemma4UnifiedParser` returns `ParsingFailed { "incomplete Gemma4 tool call" }` -> red (ERROR), while the Python parser DISPATCHES the partial call with its truncated arguments (`{city: "Par"}`) — worse for a side-effecting action, since the client executes a call the parser never finished reading.
- **P3 Empty arguments** -> `{}`.
- **P4 Structural whitespace** -> strip only tokenizer-structural whitespace bound to the marker grammar (e.g. gemma4 `thought\n`), preserve model-authored whitespace. RESOLVED for gemma4 by `ReasoningSpec::start_label`: the role label is consumed when present and TOLERATED when absent, so `<|channel>thoughtful musing<channel|>` keeps its first word and a bare `<|channel>` still opens a thought instead of leaking as text. Folding the label into the opener would have passed this corpus and broken both.
- **P5 Implicit reasoning start** -> prompt-conditioned per family (forced-reasoning models start in reasoning with no `<think>`).
- **P6 Marker quoted in prose** -> counts only as a real control token; text-only input is best-effort (known limitation, not pass/fail).
- **P7 Nested channel markers (a marker of one channel inside another)** -> marker recognition is CHANNEL-SCOPED, and both directions follow the same best-effort-recovery rule (recover real structure, never leak markup, never drop a valid call):
  - Inside a **quoted tool-argument string value**, marker-looking bytes are DATA (I7). A reasoning marker there does NOT open a reasoning channel — it is the literal arg string (`reason_markup_in_arg`). A reasoning-first pipeline that extracts `<think>`/`<|channel>` before tool parsing corrupts the arg -> red (ARG_MISMATCH / MERGE).
  - A **well-formed tool-call envelope inside a reasoning span** is STRUCTURAL: break out of reasoning, emit the call, resume reasoning after its close (`tool_in_reason`). Leaking the raw `<|tool_call>...<tool_call|>` into `reasoning_content`, or dropping the call, is the regression -> red (LEAK). The asymmetry is deliberate: quote delimiters explicitly mark a data region, whereas a reasoning span is opaque text that can still contain recoverable structure.

> P1/P2 are RESOLVED by the documented best-effort-recovery contract above (not open product questions). P4 is now resolved in code for gemma4 (optional role label) and stays a judgment call for any family that adds structural whitespace of a different shape; cases depending on it carry a `policy:` tag. P7 is RESOLVED by the same contract (no markup leak, no dropped call); the families still on the split path diverge, which is the gap it documents.

## Divergence classes (how a non-matching cell is colored)

`MATCH` (green) · `ORDER` / `MERGE` / `LOSS` (the unification gap) · `LEAK` (markup in text, `↯`) · `ARG_MISMATCH` / `WHITESPACE` (version drift) · `ERROR` (engine hard-errored where the spec expects graceful output).

The Dynamo column is a per-family mixture. The current corpus families — `deepseek_v4`, `deepseek_v41`, `qwen3`, `gemma4`, `glm47`, `kimi_k2`, `kimi_k3`, and `muse_glimmer` — run native `UnifiedParser` implementations. A future family without a native implementation falls back to the v1-reasoning + v2-tool split, and its cells must name that path explicitly.

## Quick reference — numbered taxonomy (`UNIFIED.<num>-<num>` / `UNIFIED.<letters/num>-<num>`)

New case IDs always use a numeric suffix: `<num>-<num>` for numeric groups or `<letters>-<num>` for model-specific groups such as `kimi-1`. Existing legacy IDs remain historical fixture identifiers and are translated on read; do not create new ones. The scenario slug is shown in parentheses. **Groups 1–9 mirror the tool-calling STREAM taxonomy** (`TOOLCALLING.streamv1.N`) as reasoning-free unified cases — this surface subsumes STREAM. **Group 10** is the reasoning axis (`REASONING.*`). **Group 11 is UNIQUE to unified**: reasoning↔tool ORDER that neither STREAM (no reasoning) nor REASONING (no ordered tool events) can express. **Group 12** is adversarial nesting — a marker of one channel inside another (P7). **Groups 30–39 are Guided Decoding**, divided by payload validity, surrounding markup, reasoning boundaries, and visible-answer markers. **Groups 40+ are prefilled request states.** Model-specific groups (`gemma`, `glm5`, `kimi`, `muse`) sort after every numeric group. The live per-family and total case counts come from `test_unified_case_counts_match_the_generator`; do not maintain a numeric total in prose.

### Group 1 — TC Single call
- **`1-1`** (`tool_only`) One tool call, no reasoning, no surrounding text. The tool suite's baseline.

### Group 2 — TC Multiple calls (TOOLCALLING.streamv1.2)
- **`2-1`** (`two_calls`) Two distinct calls back-to-back, order preserved. This is also covered in: TOOLCALLING.streamv1.2.a.
- **`2-2`** (`two_calls_same_name`) Two calls to the SAME function, different args — must not dedup or merge. This is also covered in: TOOLCALLING.streamv1.2.d.

### Group 3 — TC No call (TOOLCALLING.streamv1.3)
- **`3-1`** (`text_only`) Plain content, zero tool structure. No spurious call. This is also covered in: TOOLCALLING.streamv1.3. No e2e case has this shape: Qwen3.6 always emits a reasoning span, so the plain-content case is corpus-only.

### Group 4 — TC Malformed envelope
- **`4-1`** (`tool_block_never_closed_then_text`) The calls opener arrives without its closing marker and prose follows. The prose remains inside the unterminated tool envelope and is discarded at EOF; it is not visible answer text. This is applicable to DSv4.1 because its native grammar has an explicit calls envelope.
- **`4-2`** (`tool_markup_only_emits_nothing`) A calls envelope contains no invocation. Both markers are control syntax and the parser emits no event.

### Group 5 — TC Truncation / recovery (TOOLCALLING.streamv1.5)
- **`5-1`** (`truncated_tool_eof`) EOF mid-call. Golden drops the partial, keeps preceding output (P2); vLLM Rust hard-errors (`ParsingFailed`). Class ERROR.
- **`5-2`** (`tool_no_close`) Complete call body but the close marker never arrives. Most grammars recover the complete call at finish; DeepSeek V4 and V4.1 require the invoke closer and drop this malformed call. This is also covered in: TOOLCALLING.streamv1.5.a.
- **`5-3`** (`orphan_close_after_prose`) Orphan close marker after prose. Golden strips it; engines may leak. Class LEAK.
- **`5-4`** (`malformed_json_then_two_valid_calls`) Across all eight families, an unfinished native JSON/string body with explicit call closers precedes `echo(value="é")` and `echo(value="Café")`. The authored `input_chunks: [input]` delivers one `push()`, then `finish()`. Marker-aligned delivery masks the Kimi K3 defect: published `dynamo-parsers-v2` versions `0.7.13` and `0.7.14` lose the first echo; `0.7.15`, now published, preserves both. During capture qualification, the candidate moved to `0.7.15` after `0.7.14` was published; its earlier unpublished checkpoint was retired rather than relabeled as released output. K2 retains the malformed call as `{}`; Qwen3, GLM47, DeepSeek V4, and Muse retain its malformed object value as a literal string. Those five families preserve both echoes in both versions. Gemma4 returns no calls and DeepSeek V4.1 raises a JSON parse error in both versions; their goldens still require both echoes, and both remain red. Parser fixes for these two failures are outside this change.

### Group 6 — TC Empty body (TOOLCALLING.streamv1.6)
- **`6-1`** (`empty_args`) Call with `{}` arguments. Must emit the call with an empty object, not drop it. This is also covered in: TOOLCALLING.streamv1.6.a.

### Group 7 — TC Argument fidelity (TOOLCALLING.streamv1.7)
- **`7-1`** (`arg_unicode`) Non-ASCII argument value round-trips byte-exact (I7). This is also covered in: TOOLCALLING.streamv1.7.b.
- **`7-2`** (`arg_marker_in_string`) A close-marker substring INSIDE a string arg is data, preserved exactly (I7). vLLM Rust truncates. Class ARG_MISMATCH.
- **`7-3`** (`deepseek_v41_mixed_control_text_in_string`) All eight families use their native string encoding to carry mixed reasoning and tool delimiters, entity text, quotes, a backslash, a newline, and surrounding spaces. The decoded string must survive exactly. This extends `7-2` beyond a single closer; the internal scenario name and original DeepSeek V4.1 input remain unchanged for capture history.
- **`7-4`** (`arg_json_null`) The request tool schema permits JSON null. Bare parameter text `null` must produce JSON `null`; grammars with explicit types use native null syntax. Variants cover nullable type arrays, anyOf, oneOf, nullable, and const. Mixed-field probes also assert that non-nullable fields remain strings. Refs #251, #268, #269. Each schema variant has a distinct fixture ID; the popup lists every applicable result. The two categories reference the same mixed-field captures.
- **`7-5`** (`arg_string_null`) The request tool schema requires a string for the tested value. Bare parameter text `null` must remain JSON string `"null"`; grammars with explicit types use native string syntax. Variants cover non-nullable unions and intersecting sibling constraints. Mixed-field probes also assert that nullable fields become null. Refs #251, #268, #269. Each schema variant has a distinct fixture ID; the popup lists every applicable result. The two categories reference the same mixed-field captures.
- **`7-4.ref`** (`arg_json_null_ref`) GLM-only regression for PR #268: an unresolved local `$ref` points to a nullable string definition on the tool parameters root. Split-chunk XML input with bare `null` must produce JSON null.
- **`7-5.ref`** (`arg_string_null_ref`) GLM and Qwen regression: an unresolved local `$ref` points to a string-only definition on the tool parameters root. Split-chunk XML input with bare `null` must preserve string `"null"`.
- **`7-9`** (`glm_ref_object`) Shared-family compatibility control for PR #271: a local object reference converts JSON object text into an object. The ref remains unresolved in the request.
- **`7-11`** (`glm_ref_encoded_targets`) Shared-family regression for PR #271: URI percent decoding and JSON Pointer unescaping resolve definition names containing a space, UTF-8 with literal `+`, and `/` plus `~`. Bare `42` becomes an integer for each target.
- **`7-12`** (`glm_ref_json_looking_strings`) Shared-family regression for PR #271: referenced string fields preserve object-looking, array-looking, and quoted JSON text exactly, including literal quote characters. An inline string field with matching object-looking bytes is a control.
- **`7-13`** (`glm_ref_scalar_types`) Shared-family regression for PR #271: integer, number, and boolean refs produce typed scalars. A sibling integer constraint narrows a string-or-integer ref; bare `42` must become an integer. All four probes keep request refs unresolved.
- **`7-14`** (`unused_reference_graph_parameter_types`) An unused compact reference graph precedes directly typed integer, boolean, and string fields. It must not change their types or discard the complete call.
- **`7-15`** (`nullable_reference_alias_literals`) Nullable aliases to string `const: "null"` and `enum: ["null"]` retain the schema-valid string null values.
- **`7-16`** (`local_schema_id_preserves_type`) A local `$id` disables unsupported reference traversal without discarding a directly declared integer type.

### Group 8 — TC Content / narration position (TOOLCALLING.streamv1.8)
- **`8-1`** (`text_before_tool`) Visible narration precedes the call. This is also covered in: TOOLCALLING.streamv1.8.a.
- **`8-2`** (`trailing_text_after_tool`) Arbitrary prose AFTER the tool section (P1). vLLM suppresses it. Class LOSS.
- **`8-3`** (`text_sandwich`) text → call → text; both text spans survive in order. This is also covered in: TOOLCALLING.streamv1.8.c.
- **`8-4`** (`text_between_calls`) call → text → call; the inter-call prose survives (v2 recovers what v1 drops). This is also covered in: TOOLCALLING.streamv1.8.d.
- **`8-5`** (`narrated_calls`) Multiple calls with narration between each — `tool_call → text → tool_call → text → tool_call`. The agentic call/narrate/call pattern; every call and inter-call text span is its own ordered event.
- **`8-6`** (`native_quoted_control_in_response`) A native control marker inside double-quoted visible response text stays literal prose.
- **`8-7`** (`native_single_quoted_word_control`) A control marker inside a single-quoted word example stays literal prose.
- **`8-8`** (`native_quoted_control_then_call`) Quoted control text stays visible, and the following real call still dispatches.
- **`8-9`** (`native_unmatched_quote_then_call`) An unmatched quote before a real control marker cannot hide the following call.
- **`8-10`** (`native_quoted_incomplete_header`) An incomplete native tool header inside quoted response text stays literal.
- **`8-11`** (`native_single_quote_contraction_response`) A control marker inside a contraction-bearing single-quoted sentence stays literal prose.

### Group 10 — Reasoning span (`REASONING.*`)
- **`10-1`** (`reason_only`) Reasoning span, nothing else. This is also covered in: REASONING.batch.2.a.
- **`10-2`** (`reason_then_content`) Reasoning then visible content, no call. This is also covered in: e2e case-0001-chinese_arithmetic__non-stream-budget_capped.json (+ 42 more: every `reasoning/core`, `reasoning/complex` and `reasoning/history` case, `tool_none_arithmetic__*`, and the SECOND step of both `lifecycle_*` — each with its `-budget_unlimited` pair).
- **`10-3`** (`two_reason_spans`) Two reasoning spans separated by content. Batch reasoning merges them → Class MERGE. This is also covered in: REASONING.batch.6.a.
- **`10-4`** (`reason_unterminated`) Stream ends inside reasoning; open reasoning promoted at finish.
- **`10-5`** (`two_adjacent_reason_spans`) Two reasoning spans with nothing between them, then the answer. The single `reasoning_text` field every batch parser exposes can only concatenate them, so adjacent spans JOIN with a newline. The counterpart — two spans separated by a call must NOT join — is pinned by `11-2` / `11-3`: a parser that always joins invents a newline the model never emitted.

### Group 11 — Reasoning ↔ tool interleaving (UNIQUE to unified; the unification gap)
- **`11-1`** (`reason_then_tool`) Reasoning fully precedes one call. Baseline ordering.
- **`11-2`** (`reason_after_tool`) Reasoning AFTER a call, then text (Example A). Class ORDER.
- **`11-3`** (`reason_interleaved`) reason → tool → reason → tool. Class MERGE.
- **`11-4`** (`reason_tool_text_reason_tool`) reason → tool → text → reason → tool. Class MERGE.
- **`11-5`** (`interstitial_text`) reasoning → visible text → call; the middle text survives in order.
- **`11-6`** (`content_then_reason_then_tool`) Content BEFORE reasoning, then a call. Class ORDER (Dynamo hoists reasoning).
- **`11-7`** (`content_then_reason`) content → reasoning → content. Class ORDER.
- **`11-8`** (`reason_tool_reason_tool_reason`) Each call wrapped by its own thought, trailing thought too. Class MERGE.
- **`11-9`** (`reason_between_calls`) call → reasoning → call; reasoning survives BETWEEN two calls. Class MERGE.
- **`11-10`** (`text_reason_tool_text_reason_tool`) Deep well-formed interleave — text → reason → tool → text → reason → tool; user text, reasoning, and calls all mix in one stream, every segment in order. Class MERGE (batch hoists both thoughts).

### Group 12 — Adversarial nesting (a marker of one channel inside another; P7)
- **`12-1`** (`reason_markup_in_arg`) "Tool call contains reasoning" — a reasoning-channel marker sits inside a quoted tool-arg VALUE. NOT a leak: an arg value is data bound for the function, not a rendered channel, so by I7 the parser preserves it byte-exact (the gemma4 native UnifiedParser confirms the golden exactly). A reasoning-first extractor lifts it out and corrupts the arg. Class ARG_MISMATCH / MERGE.
- **`12-2`** (`tool_in_reason`) "Reasoning contains tool call" — a well-formed tool-call envelope nested inside a reasoning span. OPPOSITE of 12-1: a reasoning span is opaque text (not a quoted data region), so a real tool-call marker inside it IS structural. Golden breaks out (reason → call → reason). Engines leak the tool markup into `reasoning_content` and drop the call. Class LEAK.
- **`12-3`** (`reason_markup_in_arg_with_text`) 12-1 WITH visible narration before and after — all three channels at once (text / tool-call-with-markup-arg / text). Golden keeps text as text, the call clean, the markup byte-exact in the arg. Class ARG_MISMATCH / MERGE.
- **`12-4`** (`tool_in_reason_with_text`) 12-2 WITH visible narration before and after — text → reason → call → reason → text. Golden breaks out and keeps the surrounding text; engines leak the nested markup. Class LEAK.

### DeepSeek V4.1 applicability
- DeepSeek V4.1 uses the ordered Unified contract for native DSML calls, reasoning interleaving, guided JSON, and prefilled states. The current corpus emits 113 of the 148 taxonomy cases for this family.
- Every taxonomy scenario declared for DeepSeek V4.1 is generated. The applicable cases include `30-13`; the Guided Decoding groups `31-1` through `35-4` except `muse-1` and `35-5`; the marker-discriminating Response row `50-4`; and `40-1` through `40-6` plus `41-1` through `41-2`. The native prefilled cases `40-1`, `40-3`, and `40-4` retain explicit inputs and outputs even though other DSv4.1 rows exercise the same transitions.
- The 35 omitted cases are the 12 integral conversion variants under `7-14`, which require Qwen3-Coder or MiniMax-M2 schema conversion; `kimi-1` through `kimi-8`, which require Kimi K3 XTML syntax; `gemma-1` through `gemma-2`, which require Gemma 4 guided call-prefix syntax; `glm5-1`, which requires GLM's argument-marker grammar; `7-4.mixed_labels`, which reproduces GLM's mixed nullable fields; `7-4.ref`, `7-5.ref`, `7-9`, `7-11`, `7-12`, and `7-13`, which probe GLM schema references; `35-5`, which tests DeepSeek V4's rejected-header quote ownership; and `muse-1` through `muse-4`, which cover Muse-specific cases. The `muse-1` duplicate does not imply that quoted or malformed model output cannot occur.
