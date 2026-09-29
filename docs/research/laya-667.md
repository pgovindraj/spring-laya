# What laya#667 needs before it merges

Ticket: [spring-laya#4](https://github.com/pgovindraj/spring-laya/issues/4). Researched 2026-09-29.

PR: [NandhaKishorM/laya#667](https://github.com/NandhaKishorM/laya/pull/667), "feat(serve): add GET /v1/models and normalise dict state / structured instructions to a fixed field order".

## Short answer

- #667 is **not mergeable yet**. It has one review, CHANGES_REQUESTED, with two correctness regressions (400 → 500) and a failing repo test, and no CI has run on it.
- The `/v1/models` half **does close** the gap-table row for `TypeSafeClient.listModels()`. It collides with open laya#281, which serves `/v1/models` in the OpenAI shape, so the shape has to be settled first.
- The normalisation half makes `laya-serve` **deterministic**, but the canonical order it picks is not always "operative text first". It also silently reorders states that clients built on purpose, including chat-message lists. That is a behaviour change for every existing `laya-serve` user, and it is on by default.
- spring-ai-typesafe#11 is **still needed**. It covers Jev and other backends, `laya-serve` with the flag off or on older versions, and keeping the order the client means. The `Map.of` pattern it reports also appears in four more Spring AI classes that #11 does not name.

## 1. State of the PR

| | |
|---|---|
| State | OPEN, not draft. `mergeable: MERGEABLE`, `mergeStateStatus: BLOCKED` |
| Author | `jidechao` (commits authored as `408645320-arch`/jidechao, co-authored by Claude Code). `author_association: NONE`, a first-time contributor |
| Head | `jidechao/laya:main`, base `main` at `9d95567` (the current tip, release 0.3.21). `maintainer_can_modify: true` |
| Commits | `457a212` feat(serve): add GET /v1/models …; `b811073` fix(serve): normalise dict state and structured instructions … |
| Linked issues | none (`closingIssuesReferences: []`). The PR body references spring-ai-typesafe#11 |
| Reviewers | no review requests. One review, by `Bruce-Yii` (CONTRIBUTOR), **CHANGES_REQUESTED**, 2026-09-28 12:13Z, on `b811073` |
| Review threads | 0 inline threads. All feedback is in the review body. No conversation comments. The PR has not changed since the review (updatedAt = review time) |
| CI | **none**. `statusCheckRollup` is empty, the head SHA has 0 check runs and 0 statuses, and no `action_required` runs exist. `ci.yml`, `docker.yml`, `docs.yml` and `security.yml` all trigger on `pull_request`, so the workflows are presumably waiting for a maintainer to approve a first-time contributor's run. No maintainer (NandhaKishorM) activity yet |

Sources: `gh pr view 667 --json …`, `gh api repos/NandhaKishorM/laya/pulls/667`, `gh api …/commits/b811073…/check-runs`, the GraphQL `reviewThreads` query, and `.github/workflows/*.yml` in the laya clone.

### Remaining blockers

I verified each one locally by applying `gh pr diff 667` to `origin/main` in a scratch worktree and running `tests/test_serve.py`, `tests/test_env_docs.py` and a probe script against `create_app(router=FakeRouter())` with a `TestClient`.

1. **A malformed `questions` now returns 500 instead of 400** (reviewer's point 1, confirmed). `_normalise_questions` calls `questions.items()` before `_check_request_limits` runs its `isinstance(questions, dict)` guard (`serve.py:294`). The call also sits outside the `try … except Exception` at `serve.py:599`, so nothing is logged.

   | `questions` | main | #667 |
   |---|---|---|
   | `[]`, `[{"type":"noul"}]`, `"nope"`, `3`, `null` | 400 | 500 |

   Fix: move the normalisation block below `_check_request_limits(state, questions)`.
2. **Deeply nested state returns 500** (reviewer's point 2, confirmed). The recursive `_normalise_state` hits Python's recursion limit on bodies that `json.loads` accepts.

   | nesting depth | main | #667 |
   |---|---|---|
   | 500 | 200 | 200 |
   | 1000 | 200 | 500 |
   | 1500 | 200 | 500 |

   Fix: an explicit-stack walk. Open laya#454 made the same fix elsewhere in `serve.py`.
3. **`tests/test_env_docs.py` fails**: `FAIL every LAYA_* the package reads is documented: {'LAYA_NORMALISE_STATE': ['laya/serve.py']}`. Fix: add a row to the env table in `docs/http-api.md` (lines 23–32). `GET /v1/models` also needs an entry under `## Endpoints`. The page's "Not (yet) here" section (line 170) should be checked too.
4. **Shape collision with laya#281** (open, by `emiliano-go`, currently CONFLICTING). #281 registers `GET /v1/models` in the OpenAI shape (`{"object":"list","data":[{"id":…}]}`, and its example reads `get("/v1/models")["data"]`). Whichever PR merges second breaks the other's clients. Both SDK records are `@JsonIgnoreProperties(ignoreUnknown = true)`, so one body carrying `models` *and* `object`/`data` would satisfy both. That is for the maintainer to settle.
5. **A test that cannot fail** (reviewer's point, confirmed). With normalisation turned off in `serve.py`, `test_state_key_order_is_normalised` still passes. Only `test_structured_instructions_are_question_first` fails. The first state it posts is already in the target order, and `first == second` compares dicts, which ignores order. The reviewer also asks that the names in `test_models_lists_checkpoints` come from `DEFAULT_MODELS` rather than a literal, so they do not conflict with open laya#638, which moves the checkpoint registry into `laya.router`.

Everything else passes: 54 passed, 2 skipped (mcp, torch) and 2 failed in `test_serve.py`. The two failures, `test_a_nested_choice_label…` and `test_a_colliding_choice_label…`, fail identically on unpatched `main` in the same torch-less venv, so they are environmental and not caused by #667. The PR body's "57 passed, 1 skipped" is from the author's fuller environment.

Beyond the review, from the repo's own rules (`AGENTS.md`/`CONTRIBUTING.md`), a maintainer will want before/after numbers against real checkpoints for a change that moves probabilities. The PR body gives them for the tool-search demo only (applicability table and 4/5 matching cloud answers). It gives none for states that the new default order *reorders away from* a deliberate client order (see §2).

## 2. Does it close the gaps?

### `/v1/models`: yes, for `listModels()`

The gap table on `upstream/laya-compat:docs/client/Laya.md:71` says `GET /v1/models` is not implemented and returns 404, so `listModels()` throws `TypeSafeNotFoundException`.

- The SDK calls `GET /v1/models` (`TypeSafeApi.DEFAULT_MODELS_PATH`, `TypeSafeApi.java:54,145`) and decodes `ListModelsResponse(@JsonProperty("models") List<ModelMetadata> models)`, with a null list mapped to `List.of()`. `ModelMetadata` is `(name, @Nullable description, @JsonProperty("release_date") @Nullable releaseDate)`. Both records ignore unknown properties.
- #667 returns `{"models":[{"name":"english","description":…},{"name":"multilingual",…},{"name":"typed-decisions",…}]}` (probed: 200. `main` gives 404). With `release_date` omitted, `releaseDate()` is `null`, which is valid because the field is `@Nullable`.
- The advertised names are exactly the ones `_resolve_model` honours, so a user can send any listed name as `model`.
- It sits behind the same optional bearer check as `/v1/systemone` (tested).

Once #667 (or a shape-reconciled successor) merges, row 71 can go. Row 72 (`model` honoured only for Laya names, `TypeSafeModels` constants auto-routed) is unchanged. `JevQuickstart`, which prints `client.listModels()`, will then list Laya checkpoints instead of throwing.

Caveat: if the maintainer picks #281's OpenAI shape *instead of* adding `models`, the row reopens: `listModels()` would return an empty list (unknown `data` is ignored and a null `models` becomes `List.of()`) rather than throwing.

### Field-order sensitivity: deterministic, but not "operative field first"

What it normalises (`_systemone_inner`, before `Router.predict`, default on, `LAYA_NORMALISE_STATE=0` turns it off):

- **dict state**: scalar-valued keys first, then container-valued keys, each group sorted by key, recursively. Dicts nested inside lists get the same treatment, and list element order is kept.
- **structured (dict) instructions**: `question` first, the rest sorted. Plain-string instructions pass through.
- **criteria**: untouched, on purpose. Option order is positional in the scoring head (`router.py:_question_schema` docstring).

Why it matters: `laya.common.serialize_state` is `json.dumps(state, ensure_ascii=False)` with insertion order kept, so the prompt is positional (`laya/common.py:74–77`).

So arrival-order randomness (Java `Map.of`) is gone **for `laya-serve`**, and shuffled orders now answer identically. The canonical order is sometimes *not* the one the PR's rationale describes ("the request text precedes the collections it is judged against"). It only puts request text first when the other fields are containers. Probed on #667:

| client state (as sent) | reaches the router as |
|---|---|
| `{"question","answer","context","tool_calls"}` (Jev judge layout the PR cites) | `{"answer","question","context","tool_calls"}`: answer before question |
| JevToolIndex single-candidate `{user_request, tool}` (both strings, `JevToolIndex.java:183`) | `{"tool","user_request"}`: request **last** |
| JevDocumentFilter/Reranker `{query, passage}` (`JevDocumentFilter.java:153`, `JevDocumentReranker.java:115`) | `{"passage","query"}`: query **last** |
| JevToolIndex catalogue entries `{"name","does"}` (a deliberate `LinkedHashMap`, `JevToolIndex.java:240–242`) | `{"does","name"}` |
| list state `[{"role","content"}, …]` (chat messages) | `[{"content","role"}, …]`: content before role, `tool_calls` last |
| `{"user","messages":[…]}` | `{"user","messages":[{"content","role"},…]}` |
| a string state (e.g. JSON the client already serialised) | unchanged |

**List states** keep element order, but every dict inside them is key-sorted. **String states** and string instructions are untouched, so any order baked into a string stays as the client wrote it.

The PR measured the effect only on the multi-candidate tool-search request, where the scalar-first rule happens to put `user_request` first. The table shows that the same rule puts the operative field last for two-scalar states, which the Spring AI RAG filter/reranker and the single-candidate tool path both send. Whether that costs accuracy has not been measured. It is worth asking for before/after numbers on those shapes, or for a rule that does not depend on the alphabet.

### Behaviour change for existing `laya-serve` users

- **On by default.** Every dict state, every dict nested in a list state, and every structured instruction is reordered. A client that already sends a deliberate order loses it and gets different probabilities from the same request. That includes callers following `Laya.md:105–106` ("build states in a fixed order (`LinkedHashMap`, not `Map.of`)"), hs-jev, and chat-shaped states. The only way out is the server-wide `LAYA_NORMALISE_STATE=0`. There is no per-request opt-out.
- **Only `laya-serve` changes.** The Python `Router`/`Agent`, CLI, MCP server, ONNX and TS surfaces still serialise in arrival order. The same state can now answer differently over HTTP and in-process.
- **Error regressions** (malformed `questions` 400 → 500, deep state 200 → 500), as listed above.
- `GET /v1/models` is purely additive. It was 404 before.

## 3. Overlap with spring-ai-typesafe#11

[spring-ai-typesafe#11](https://github.com/spring-ai-community/spring-ai-typesafe/issues/11) is open, filed 2026-09-28 by the same author (`jidechao`), and has one comment (`bengbengbalabalabeng` asking whether the hosted TypeSafe API is order-sensitive too, unanswered). It asks `JevToolIndex` to build its state and applicability instructions with `LinkedHashMap`, operative field first. Its own note says the laya-side fix exists and that it files the issue "because the client-side fix benefits the hosted API and other backends as well". No PR for it exists yet.

If #667 merges, #11 is **still needed**:

- **Jev and other backends.** #667 changes nothing off `laya-serve`. Whether hosted Jev is order-sensitive is unknown. That is the open question on #11. Determinism across JVM runs is still worth having whatever the answer.
- **The client's intended order vs. Laya's canonical order.** #11 puts `user_request` first and `question` first. #667 agrees on those two for the multi-candidate call, but reorders the single-candidate `{user_request, tool}` state to request-last. With #11 fixed client-side, `laya-serve` users can turn `LAYA_NORMALISE_STATE=0` on and keep the client's order.
- **Older `laya-serve` versions**, and deployments that disable the flag.
- **Field order inside strings.** Nothing server-side can touch it, and #667 passes string state and string instructions through unchanged. Today no Spring AI class serialises a map into a string before sending, so this is theoretical for the integrations, but it applies to user code.
- **#11 is narrower than the pattern.** The same `Map.of` for state or instructions appears in `JevDocumentFilter` (state `:153`, instructions `:240`), `JevDocumentReranker` (state `:115`, instructions `:195`) and `JevChatModel` (every chat message `Map.of("role",…,"content",…)`, `:350`). `JevGuardrail`'s `Map.of(TEXT_FIELD, text)` has one key and is harmless. A backend-neutral upstream fix should cover all of them, not just `JevToolIndex`. That also makes it a stronger PR for @tzolov on its own merits.

The two fixes complement each other: #667 protects `laya-serve` from any unordered client, and #11 (widened) makes the Spring client send a deliberate, stable order to every backend. Once both land, the field-order bullet in `Laya.md:105–106` can shrink to a note about `LAYA_NORMALISE_STATE`.

## Sources

- `gh pr view 667 -R NandhaKishorM/laya` (body, commits, review), `gh pr diff 667`, `gh pr checks 667`, the GraphQL `reviewThreads` query, `gh api …/pulls/667`, and `…/commits/b811073…/check-runs` and `/status`
- `gh pr view 281`, `gh pr diff 281`, `gh pr view 638`, `gh pr view 454` on NandhaKishorM/laya
- laya clone at `9d95567`: `laya/serve.py`, `laya/common.py:74–77`, `laya/router.py:134–141`, `docs/http-api.md`, `tests/test_env_docs.py`, `.github/workflows/`
- Local verification: #667 applied to `9d95567`, `pytest tests/test_serve.py`, `python tests/test_env_docs.py`, and a TestClient probe (malformed `questions`, nesting depth, list/judge/string states, `/v1/models`)
- `gh issue view 11 -R spring-ai-community/spring-ai-typesafe`
- spring-ai-typesafe `upstream/laya-compat` (`defd929`): `docs/client/Laya.md:71,105–106`, `TypeSafeApi.java:54,145`, `TypeSafeClient.java:311`, `response/ListModelsResponse.java`, `response/ModelMetadata.java`
- spring-ai-typesafe `upstream/main` (`5c469ce`): `JevToolIndex.java:146,183,237–255`, `JevDocumentFilter.java:81–84,153,240`, `JevDocumentReranker.java:80–83,115,195`, `JevChatModel.java:350`, `JevGuardrail.java:174`
