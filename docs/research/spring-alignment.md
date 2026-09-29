# Getting Laya supported in Spring: alignment research

Researched 2026-09-29. Primary sources only. Upstream code references are pinned to
`spring-ai-community/spring-ai-typesafe` commit `5c469ce` (the `main` branch on that date), abbreviated below as
`TS@5c469ce`. The base URL for those links is `https://github.com/spring-ai-community/spring-ai-typesafe/blob/5c469ce/`.

## TL;DR

**Don't create a new project. Contribute to `spring-ai-typesafe`.** Its maintainer already treats Laya as a supported backend, so the work is to help finish what he started.

- The maintainer, Christian Tzolov (@tzolov), has already done three things:
  - opened a **draft PR that documents Laya** and lets the live integration tests follow `TYPESAFE_BASE_URL` ([spring-ai-typesafe#9](https://github.com/spring-ai-community/spring-ai-typesafe/pull/9));
  - opened an **issue asking whether to rename `Jev*` types to model-neutral `SystemOne*`**, citing Laya ([#7](https://github.com/spring-ai-community/spring-ai-typesafe/issues/7));
  - told a Spring AI core requester that the integrations "aren't tied to one model: the open-source Laya serves the same API" ([spring-projects/spring-ai#7018 comment](https://github.com/spring-projects/spring-ai/issues/7018#issuecomment-5795942139)).
- **Laya already works today** if you set `spring.ai.typesafe.base-url` and a dummy `api-key`. PR #9 says: "Against Laya, the SDK handled every response without changes; all failing ITs are protocol gaps or answer-quality differences, not SDK bugs."
- **The remaining gaps are small. Fix them on both sides:**
  - **Laya side, which we control:**
    - add `GET /v1/models` (already open as [laya#667](https://github.com/NandhaKishorM/laya/pull/667));
    - accept nouls that have criteria but no instructions;
    - consider accepting `null` score levels.
  - **SDK side, as small focused PRs:**
    - honour `Retry-After` on 503;
    - make the API key optional for non-hosted base URLs;
    - fix the ordered-map bug ([#11](https://github.com/spring-ai-community/spring-ai-typesafe/issues/11));
    - document that confidence thresholds depend on the backend.
- **Why not the alternatives:**
  - A separate `spring-ai-laya` community project would duplicate this code and join a slow queue: 13 project requests are open, many with no reply since April 2026 (see §3b).
  - A Spring AI core integration isn't realistic: the core team sends non-OpenAI-compatible models to the community org (see §3c).
- **How to use `pgovindraj/spring-laya`:** make it a *mirror of upstream history* (an `upstream` remote, with topic branches cut from `upstream/main`) so each change can become a PR with no churn. Put Laya-only extras (for example a Testcontainers module or benchmarks) in a separate module there.
- **Next step:** comment on #7 and #9 before writing code.

---

## 1. Contribution rules

### 1.1 spring-ai-typesafe (the target repo)

- **No CONTRIBUTING file.** The repo has none, and neither does the org-level `.github` repo, which contains only `CODE_OF_CONDUCT.md` and `profile/README.md` ([spring-ai-community/.github](https://github.com/spring-ai-community/.github)). GitHub Discussions is **disabled** on the repo (`has_discussions: false` from `gh api repos/spring-ai-community/spring-ai-typesafe`). This is despite the community governance saying that "each project uses GitHub Discussions" ([Governance wiki](https://github.com/spring-ai-community/community/wiki/Governance)). In practice, issues are the channel.
- **The Spring AI guidelines apply by convention.** The community's project-creation guide says projects "can use existing contribution guidelines such as [spring-framework's CONTRIBUTING]" ([Project Creation Guide](https://github.com/spring-ai-community/community/wiki/Project-Creation-Process)). The main Spring AI rules are ([spring-ai CONTRIBUTING.md](https://github.com/spring-projects/spring-ai/blob/main/CONTRIBUTING.md)):
  - **DCO, not CLA.** "All commits must include a Signed-off-by trailer … Developer Certificate of Origin" (see also [Hello DCO, Goodbye CLA](https://spring.io/blog/2025/01/06/hello-dco-goodbye-cla-simplifying-contributions-to-spring)).
  - **Commit titles:** start with an uppercase verb and use no `fix:`/`feat:` prefixes. Our Laya repo's conventional-commit style (`feat(agent):`) must **not** be carried over.
  - **Issues first?** "No, just create the pull request … You only have to create an issue if you want to start a discussion first." Don't send PRs "for straightforward or polish-style changes".
  - **AI assistance:** it is accepted but "must be carefully reviewed by a human who remains accountable". Accounts run by autonomous bots are banned.
  - **Style:**
    - Spring Framework checkstyle conventions;
    - tabs, LF line endings, code wrapped at 120 characters, Javadoc at 90;
    - import groups ordered `java.*` / `javax.*`+`jakarta.*` / others / `org.springframework.*` / static imports;
    - no wildcard imports, and no static imports in production code;
    - an Apache license header on every file;
    - `@since` on new public API;
    - `.md` files written one sentence per line.
- **What spring-ai-typesafe actually does**, from its history:
  - **Sign-offs:** every commit on `main` carries `Signed-off-by: Christian Tzolov <christian.tzolov@broadcom.com>`, **including squash-merged community PRs** such as #2 by Ashfaqbs. The only exception is #10. So the maintainer adds the sign-off at merge time, and no DCO bot enforces it (`git log --format='%(trailers:key=Signed-off-by)'` on the clone). Add your own `Signed-off-by` anyway; it costs nothing.
  - **Commit titles:** plain imperative sentences, with the squash-merge adding `(#N)`, e.g. "Allow Noul questions without instructions when criteria are given (#2)".
  - **Formatting is by convention only.** There is **no spring-javaformat, checkstyle or license plugin** in the build (`grep` of `pom.xml` at TS@5c469ce). The sources still follow Spring style by hand: tab indentation, Spring import grouping, `@author` tags, JSpecify `@Nullable` (`typesafe-java-sdk/src/main/java/org/springaicommunity/typesafe/TypeSafeClient.java`).
  - **License header:** `Copyright 2026 - 2026 the original author or authors.` + Apache-2.0 (every source file, e.g. `typesafe-java-sdk/.../JevBatchOptions.java`).
  - **Tests:** the offline suite uses `MockRestServiceServer`. `*IT` classes sit behind the `-Pintegration-tests` profile *and* `@EnabledIfEnvironmentVariable` on `TYPESAFE_API_KEY` ([README "Building"](https://github.com/spring-ai-community/spring-ai-typesafe/blob/5c469ce/README.md)). CI runs `mvn verify` on JDK 17 for pull requests (`.github/workflows/ci.yml`).
  - **Docs:** MkDocs Material under `docs/`, with nav in `mkdocs.yml` and release notes in `docs/changes.md`. Docs are published with `mike` (`.github/workflows/docs.yml`). A feature PR is expected to include its docs page; PR #9 adds `docs/client/Laya.md` plus a `mkdocs.yml` nav entry.
  - **Toolchain:**
    - Java 17, Spring AI 2.0.1, Spring Boot 4.0.7 (`pom.xml` lines 55–61);
    - groupId `org.springaicommunity`, root package `org.springaicommunity.typesafe`;
    - current version `0.3.0-SNAPSHOT`; releases so far `v0.1.0` (2026-09-20) and `v0.2.0` (2026-09-23) (`gh release list`).

### 1.2 spring-ai-community org (only relevant if proposing a new project)

- **Governance** ([wiki](https://github.com/spring-ai-community/community/wiki/Governance)):
  - contributors keep their IP, and everything is Apache-2.0;
  - project Technical Leads have "full authority over technical decisions within their project";
  - Community Leads are **Thomas Vitale, Sergi Almar and Adib Saikali**;
  - a new project needs "unanimous approval from all Spring AI Community Leads", which should come "within 2 weeks of submission when possible".
- **Process:** fill in the issue form ([application.yml](https://github.com/spring-ai-community/community/blob/main/.github/ISSUE_TEMPLATE/application.yml)). It requires:
  - a POC repo URL and docs;
  - how the project integrates with Spring AI;
  - the maintainer team;
  - checkboxes for unit and integration tests, Apache-2.0, a public repo, the code of conduct, contribution guidelines and semver;
  - a packaging choice (`io.github.spring-ai-community` groupId, or your own publishing);
  - a declaration of any commercial single-vendor ownership.

  Requirements are in the [Project Creation Guide](https://github.com/spring-ai-community/community/wiki/Project-Creation-Process). Projects should be named `spring-ai-[key-feature]`, versions 0.x mean not yet stable, and "Commercial ownership/control is acceptable".
- **Official listing:** being listed on spring.io and in the docs requires quarterly health review, CI, docs "that follow Spring AI's format", and "compatibility with the latest Spring AI release". Community projects are **not** in the Spring AI BOM ([Referencing Community Projects](https://github.com/spring-ai-community/community/wiki/Spring-AI-Referencing-Community-Projects)).

## 2. Scope fit

### 2.1 Is Laya / self-hosted backend support in scope?

Yes. There is direct evidence that the maintainer wants it:

1. **Draft PR #9, "Let the live ITs follow TYPESAFE_BASE_URL and document using Laya"**, opened by @tzolov on 2026-09-23 from branch `laya-compat` and still a draft ([#9](https://github.com/spring-ai-community/spring-ai-typesafe/pull/9)). It:
   - adds `docs/client/Laya.md`, which covers running `laya-serve`, pointing the client and starter at it, a table of protocol differences measured on Laya 0.3.11, and how answers compare with Jev;
   - removes the pinned `DEFAULT_BASE_URL` from five IT classes;
   - adds commented-out Laya settings to `examples/src/main/resources/application.properties`;
   - calls Laya "a stand-in, not a replacement", useful "for offline development, local experiments and CI without an API key".
2. **Issue #7 (naming)**, by @tzolov. The integrations "depend on the System One API contract (`POST /v1/systemone`), not on Jev itself … On Laya, a name like `JevJudge` would be misleading." It lists as an open follow-up: "should the SDK document or test against other System One servers, such as `laya-serve`, via the base URL?" ([#7](https://github.com/spring-ai-community/spring-ai-typesafe/issues/7)). No comments yet, so it's an open invitation.
3. **The Spring AI core reply**: @tzolov on [spring-projects/spring-ai#7018](https://github.com/spring-projects/spring-ai/issues/7018#issuecomment-5795942139) wrote "The integrations talk to the System One API … so they aren't tied to one model: the open-source Laya serves the same API."
4. **Community users already run it against Laya.** [#11](https://github.com/spring-ai-community/spring-ai-typesafe/issues/11) (@jidechao) reports randomised JSON key order from `Map.of` in `JevToolIndex`, found using `laya-serve`. The same author opened [laya#667](https://github.com/NandhaKishorM/laya/pull/667), which adds `GET /v1/models` and normalises field order on the server.
5. **Heads-up: #7 and the #7018 comment link Laya as `github.com/receptron/laya`.** That repo is a *Node.js/ONNX runner* ("Run Laya … from Node.js / TypeScript via ONNX Runtime", `gh repo view receptron/laya`). The server is in `NandhaKishorM/laya`, as PR #9 correctly links. Offering this correction is an easy, welcome first contribution.

**Does the TypeSafe company control the repo?** No evidence that it does:

- the repo lives in the community org;
- the author commits as `christian.tzolov@broadcom.com` (git log);
- the contributors are tzolov, Ashfaqbs and xujiantop-crypto (`gh api .../contributors`);
- the launch blog calls it "a new Spring AI Community project that integrates TypeSafe AI's hosted Jev API" ([spring.io blog, 2026-09-21](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment)).

Two things to respect. The *artifact names* stay TypeSafe-branded: issue #7 says `TypeSafeClient`, the artifact IDs, the package root and the `JEV_*` constants stay. And the hosted Jev API stays the default (`DEFAULT_BASE_URL = "https://api.typesafe.ai"`, `TypeSafeConstants.java:42`).

### 2.2 Does laya-serve work with just `base-url` + a dummy `api-key`?

Mostly yes. This is confirmed by PR #9 and by reading TS@5c469ce. Here are the code paths where it rubs:

| # | Code path (TS@5c469ce) | Behaviour vs laya-serve | Breaks? |
|---|---|---|---|
| 1 | Starter `@ConditionalOnProperty(prefix="spring.ai.typesafe", name="api-key")` plus a runtime non-blank check (`spring-ai-starter-typesafe/.../TypeSafeAutoConfiguration.java:48,64-68`) | Without `api-key`, **no `TypeSafeClient` bean is created at all**. Laya's auth is optional, yet a key must be set. Any value works when `LAYA_API_KEY` is unset; it must match when set (PR #9 doc) | Workaround: dummy key |
| 2 | `TypeSafeClient.Builder.apiKey(String)` → `Assert.hasText`; `build()` falls back to `TYPESAFE_API_KEY` and fails if it is empty (`TypeSafeClient.java:470-472, 567-573`); `TypeSafeApi` always sends a bearer token (`api/TypeSafeApi.java:65-78`) | Same as above for plain Java | Workaround: dummy key |
| 3 | `listModels()` → `GET /v1/models` (`TypeSafeClient.java`, `api/TypeSafeApi.java`). Used only by `examples/.../JevQuickstart.java`, not by any integration in `typesafe-spring-ai` (grep) | 404 → `TypeSafeNotFoundException` | Only `listModels()`. Fixed by [laya#667](https://github.com/NandhaKishorM/laya/pull/667) |
| 4 | Criteria-only nouls are allowed since #2: the builder requires "instructions or criteria (whenTrue/whenFalse)" ([commit 39f66fd](https://github.com/spring-ai-community/spring-ai-typesafe/commit/39f66fd)) | Laya returns **422** (PR #9 table). Every noul built *inside* `typesafe-spring-ai` sets `.instructions(...)` (`JevGuardrail.java:324`, `JevDocumentFilter.java:221-239`, `JevDocumentReranker.java:194`, `JevToolIndex.java:185,249`). Only **user-written** criteria-only nouls, e.g. `JevJudge` criteria, break | User code only. Best fixed in Laya |
| 5 | Score levels: `Score.Builder.level(JsonContent)` only asserts non-null, so `JsonContent.NULL` gets through (`question/Score.java:147-162`) | Laya rejects null levels | Edge case, user code |
| 6 | Retry: `RetryPolicy.isRetryable` retries 408, 429 and **any status ≥ 500** (`RetryPolicy.java:62,94-101`). But `backoffFor` honours `Retry-After` **only for `TypeSafeRateLimitException` (429)** (`RetryPolicy.java:111-117`). The error handler maps only 429 to that type (`api/TypeSafeResponseErrorHandler.java:100-101`) | Laya's 503 is retried, with exponential backoff from 500 ms capped at 5 s, 2 retries by default (`RetryPolicy.java:34,142`), **ignoring its `Retry-After`** | Works, but not ideal |
| 7 | Server `confidence`: `JevConfidenceGate.decide(ChoiceAnswer)` uses `answer.confidence()` against `DEFAULT_FLOOR = 0.6` (`judge/JevConfidenceGate.java:59,93-107`) | Laya computes confidence as 1 − normalised entropy, not Jev's formula, so the same floor means something different. By contrast **`JevJudge` computes its own "support" from `probabilities()`** and uses `confidence()` only when probabilities are missing (`judge/JevJudge.java:408-455`), so the judge is mostly backend-neutral | Needs recalibration, not a crash |
| 8 | Thresholds tuned on Jev: `JevGuardrail` review/action 0.35/0.70 (`advisor/JevGuardrail.java:302-304`), `JevToolIndex` applicability 0.5 (`toolsearch/JevToolIndex.java:283`) | PR #9 found Laya's answers "less decisive" and got 2 of 4 on tool search | Needs recalibration |
| 9 | `Map.of` used for state and structured instructions (13 uses in `typesafe-spring-ai/src/main`; `JevToolIndex` detailed in #11) | Laya is sensitive to field order, so answers change between JVM runs | Yes, flaky. Fix on both sides (#11 client, laya#667 server) |
| 10 | Default model `jev-latest` (`TypeSafeConstants.java:45`) | Laya auto-routes names it doesn't know (PR #9 table) | Harmless |
| 11 | `requestId()`, `usage().outputTokens()`, `errorType()` | null / 0 / null (PR #9 table) | Only the ITs that assert them |
| 12 | Extra Laya fields (`answer_confidence`, `routing`, `action`) | Ignored by the client (PR #9) | No, but not exposed either |
| 13 | Choice options: no client-side limit | Laya allows at most 100 | Edge case |

## 3. Options

### (a) Small PRs making spring-ai-typesafe backend-neutral: **recommended core**

**Pros:**
- It matches the maintainer's own stated direction (#7, #9, the #7018 reply).
- The code already sits on Spring AI SPIs (`CallAdvisor`, `DocumentPostProcessor`, `ToolIndex`, `Evaluator`, `ChatModel`, per the #7018 reply).
- It is published to Central, with docs and a Spring blog post.
- Laya users get all of it for free.

**Cons:**
- The TypeSafe branding stays (per #7, artifact and package names stay).
- The maintainer controls the pace. #9 has been a draft since 2026-09-23, and #3 and #11 have no maintainer reply.
- Laya-specific features such as `routing`, `answer_confidence` or `action.act_probability` probably don't belong in a Jev-first SDK, unless they're exposed as a generic "extra fields" map.

Candidate PRs, each small and one logical change, as Spring AI's CONTRIBUTING asks:

1. **Honour `Retry-After` on any retryable status (503, and also 529)**, not only 429. It's generic HTTP correctness, not Laya-specific. This touches `RetryPolicy.backoffFor` and `TypeSafeResponseErrorHandler`.
2. **Starter: create the client when `base-url` is set even without `api-key`**, and let the builder skip the `Authorization` header when no key is configured and the base URL isn't the hosted one. Raise it as an issue first: it changes the fail-fast behaviour that `TypeSafeAutoConfiguration.java:64-68` protects on purpose.
3. **Use `LinkedHashMap` in `JevToolIndex` and the other `Map.of` states** (#11 already has the diagnosis and the fix). It helps determinism on every backend.
4. **Help land #9**: re-measure the "What differs" table against current Laya (0.3.21; the table says 0.3.11), and update it once laya#667 and the criteria-only-noul fix ship.
5. **Confidence:** start with a docs note that `JevConfidenceGate` floors depend on the backend. Possibly add a backend-neutral helper that recomputes confidence from `probabilities()`, the way `JevJudge` already derives "support". A pluggable SPI is probably too much for now.
6. **Support #7's model-neutral naming**, option 1 (`SystemOne*`), with Laya as the concrete second implementation. It's the maintainer's call, but a comment with a use case helps.

### (b) New spring-ai-community project (`spring-ai-laya`): **not recommended**

- **Process:** the issue form plus unanimous Community Lead approval (§1.2).
- **Evidence that it's slow:** of the requests opened since 2026-04, most are open with **0 comments**: #28, #29, #30, #32, #34, #35, #36, #37, #38, #39. #31 notes "the lengthy process of inclusion" and published under its own groupId meanwhile ([community#31](https://github.com/spring-ai-community/community/issues/31)). #40 (2026-09-02) has one lead "looks great" and is still asking for a review ([community#40](https://github.com/spring-ai-community/community/issues/40)). Historically @markpollack onboarded projects, e.g. [#16 Replicate](https://github.com/spring-ai-community/community/issues/16), onboarded after about 5 weeks. Wiki governance promises "within 2 weeks … when possible".
- **Duplication:** it would copy the same `/v1/systemone` client and integrations that #7018 already points people to. That fragments the ecosystem and invites the "why not contribute there?" question. There's precedent: #3's independent `jev-spring-boot-starter` author ended up offering to fold into spring-ai-typesafe ([#3](https://github.com/spring-ai-community/spring-ai-typesafe/issues/3)).
- **When it would make sense:** only if Laya needs a surface the System One protocol can't express, such as local in-JVM inference via ONNX or embedded Laya without HTTP, and the SDK maintainer declines it.

### (c) Upstream Spring AI (spring-projects/spring-ai) model integration: **not plausible now**

- Spring AI's model abstractions are chat, embedding, image and so on. A reranking/scoring abstraction is still just a request ([spring-ai#5776](https://github.com/spring-projects/spring-ai/issues/5776), `status: waiting-for-triage`).
- The stated policy for new models: use the OpenAI starter if compatible, "otherwise … incubating the support within the Spring AI Community, and once it matures, integrating it into the main project" (@sunyuhan1998, contributor, [spring-ai#4296](https://github.com/spring-projects/spring-ai/issues/4296)). A vLLM integration request was likewise redirected to the community org ([spring-ai#3882](https://github.com/spring-projects/spring-ai/issues/3882)).
- The System One request to core was answered by pointing to spring-ai-typesafe ([spring-ai#7018](https://github.com/spring-projects/spring-ai/issues/7018)).
- Community projects are also kept out of the Spring AI BOM ([wiki](https://github.com/spring-ai-community/community/wiki/Spring-AI-Referencing-Community-Projects)).

### (d) Mix: **the recommendation**

Combine (a) with **Laya-side parity work** and a **thin companion repo** (`pgovindraj/spring-laya`) that holds only what doesn't belong upstream. Examples: a Testcontainers module that starts `laya-serve`, a `laya` Spring profile or example, and threshold calibration or benchmark harnesses, all depending on `typesafe-bom`. If one of these proves broadly useful (for example Testcontainers for any System One server), offer it upstream later.

**Ordered steps:**

1. **Talk first, no code yet:**
   - comment on [#7](https://github.com/spring-ai-community/spring-ai-typesafe/issues/7): say you're a Laya maintainer, support option 1, answer the "open follow-up" with "yes, and we'll help";
   - comment on [#9](https://github.com/spring-ai-community/spring-ai-typesafe/pull/9): offer to re-measure on current Laya, point out the `receptron/laya` vs `NandhaKishorM/laya` link, and ask which of the gaps he'd take PRs for.
2. **Close gaps in Laya** (our repo, our rules):
   - merge or land laya#667 (`/v1/models`, field-order normalisation);
   - accept criteria-only nouls (return what Jev returns);
   - decide on null score levels;
   - consider also sending `retry-after-ms`.
   - Every gap closed here is one fewer SDK PR.
3. **SDK PRs, one at a time**, in the order Retry-After → #11 ordered maps → optional API key (issue first) → confidence docs. Follow §1.1 style.
4. **Stand up `spring-laya`** for Laya-only extras (§4), and keep Laya's CI running the upstream `*IT` suites against `laya-serve`. The commands are in PR #9's "Integration tests against Laya" section.
5. **Only if upstream declines something essential:** propose a community project using the issue form, with `spring-laya` as the POC.

## 4. Developing in `pgovindraj/spring-laya` while staying aligned

**Fork vs. separate repo.** GitHub PRs need a fork relationship, and `spring-laya` was created empty, not as a fork. There are two workable set-ups; the second is recommended.

- **Upstream-bound code**, i.e. changes to spring-ai-typesafe itself: use a real GitHub fork (`gh repo fork spring-ai-community/spring-ai-typesafe`) and cut one topic branch per PR from `upstream/main`. Or add upstream as a remote in `spring-laya` and push those branches to the fork when ready. Never mix Laya-only files into these branches.
- **Laya-only code** lives in `spring-laya` as its own Maven project that *consumes* the upstream artifacts via `typesafe-bom`.

**Match upstream so upstreaming is a copy, not a rewrite:**

- **Toolchain:** Java 17 release target, Spring Boot 4.0.7, Spring AI 2.0.1, Maven wrapper, JSpecify `@Nullable` (`pom.xml`; `typesafe-java-sdk/pom.xml:70`).
- **Coordinates:** groupId `org.springaicommunity`. Packages go under `org.springaicommunity.typesafe.<area>` for anything meant to move upstream. Use a distinct subpackage such as `org.springaicommunity.typesafe.laya` or a separate root for Laya-only code, so no class names clash.
- **Headers:** the exact upstream Apache header `Copyright 2026 - 2026 the original author or authors.`; `@author` on types, `@since` on new public API (Spring AI CONTRIBUTING).
- **Formatting:** tabs, 120 columns, Javadoc at 90, Spring import order. There's no formatter upstream to rely on. Optionally run `spring-javaformat` **locally only**, check that it produces no diff on upstream files, and don't add it to upstream poms.
- **Tests:** mirror upstream's split. Offline tests use `MockRestServiceServer`; `*IT` tests go behind `-Pintegration-tests` and an env-var guard (`LAYA_BASE_URL` or reuse `TYPESAFE_BASE_URL`, as PR #9 does).
- **Docs:** MkDocs Material pages in upstream's layout (`docs/<area>/<Type>.md` and a `mkdocs.yml` nav), so a page can move straight into `docs/client/`.
- **Commits:** Spring style (imperative title, no prefix) with `Signed-off-by`, even though the Laya repo itself uses conventional commits.

**Tracking upstream:**
- Watch releases on spring-ai-typesafe (releases are frequent: 0.1.0 on 09-20 and 0.2.0 on 09-23).
- Pin `typesafe-bom` to the latest release, and have one CI job run against `0.3.0-SNAPSHOT` from the Central snapshot repository (`publish-snapshot.yml` publishes on every push to `main`).
- Rebase topic branches on `upstream/main` before opening PRs.

## 5. Who to contact (public info only)

- **Christian Tzolov, [@tzolov](https://github.com/tzolov)**: sole maintainer and author of spring-ai-typesafe (commit history), and author of #7, #9 and the #7018 reply. Spring AI team at Broadcom (commit email domain). Reach him through issues or PRs on [spring-ai-typesafe](https://github.com/spring-ai-community/spring-ai-typesafe/issues); Discussions is disabled.
- **Spring AI Community Leads:** Thomas Vitale ([@ThomasVitale](https://github.com/ThomasVitale)), Sergi Almar ([@salmar](https://github.com/salmar)), Adib Saikali ([@asaikali](https://github.com/asaikali)) ([Governance](https://github.com/spring-ai-community/community/wiki/Governance)). They're only relevant for a new-project request or escalation. @markpollack has handled onboarding in practice ([community#16](https://github.com/spring-ai-community/community/issues/16)).
- **Allies already bridging Laya and Spring:**
  - [@jidechao](https://github.com/jidechao): #11, laya#667;
  - [@commandodev](https://github.com/commandodev) (Ben Ford): author of `laya-serve`, [laya#31](https://github.com/NandhaKishorM/laya/pull/31).
- **Broader channels:**
  - [Spring AI GitHub Discussions](https://github.com/spring-projects/spring-ai/discussions) and the Stack Overflow `spring-ai` tag (per Spring AI CONTRIBUTING);
  - the [Spring AI Community site](https://springaicommunity.mintlify.app/);
  - [@SpringAICentral](https://twitter.com/SpringAICentral) and [Bluesky](https://bsky.app/profile/spring-ai.bsky.social) (org profile README).
  - Governance mentions a Slack for project leads only.
