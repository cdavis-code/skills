---
name: postforge-publishing
description: Turn what an expert knows into a LinkedIn post with PostForge, attaching a source as evidence. Use when the user asks to draft, review, or schedule a LinkedIn post with PostForge ("generate a post for review around my topics", "find an article and draft a LinkedIn post", "draft a post from my expertise", "post this on Monday"). Enforces the human-in-the-loop contract: nothing publishes without an explicit approval instruction.
---

# PostForge Publishing

PostForge turns what the expert knows into a LinkedIn post, with a source attached as evidence. There are two starting points: **from an article** (react to a source you or PostForge found) and **from your expertise** (start with what you know, then attach a source). Both go through the **postforge MCP server**. The server is hosted inside the PostForge API itself at `POST /mcp` (MCP Streamable HTTP), and its tools call the same code paths the web app uses, so every validation, credit rule, and human-in-the-loop (HIL) pause behaves exactly like the app.

## Setup

Mint an API token in the app under **Settings → MCP access** (create, then copy the `pf_...` value — it is shown once). A client that cannot send an `Authorization` header can connect with OAuth instead: the server publishes its metadata, registers the client, shows a consent page where the user picks the scope, and issues the same kind of `pf_...` token. Both kinds appear in the same list and stop working the moment one is revoked. Headless alternative (admin-only), replacing `<tenant-id>` with the tenant from the `tenants` table:

```bash
cd saas/api && .venv/bin/python -c "
import asyncio, uuid
from app.db.session import SessionLocal
from app.mcp import tokens

async def main():
    async with SessionLocal() as s:
        plaintext, _ = await tokens.issue_token(s, uuid.UUID('<tenant-id>'), name='claude-code')
        print(plaintext)

asyncio.run(main())
"
```

The token prints once (`pf_...`); only its hash is stored, so it can never be shown again. It acts as the user — including approving and publishing posts.

**Connect your harness.** PostForge is a remote Streamable HTTP MCP server: add the deployed origin (`https://<your-domain>/mcp`; local: `http://localhost:8000/mcp`) with an `Authorization: Bearer pf_...` header in your harness's MCP settings. The repository README's **Install in your harness** section has the file or command for Claude Code, Qoder, Cursor, VS Code/Copilot, and Codex, plus the `mcp-remote` bridge for harnesses that only support stdio.

If the `postforge` tools are not in your tool list, the server is not connected yet — installation is the step before this skill can run. Never paste the token into chat or commit it to the repository. A `draft-only` token covers drafting and review; `full` is only for approving or publishing.

A generic `mcpServers` entry:

```json
{
  "mcpServers": {
    "postforge": {
      "type": "http",
      "url": "http://localhost:8000/mcp",
      "headers": { "Authorization": "Bearer pf_<token>" }
    }
  }
}
```

Dev alternative: skip the token and send the browser's session cookie instead — `"headers": { "Cookie": "session=<value>" }` (DevTools > Application > Cookies). The endpoint accepts it **read-only**: every write tool is rejected with "Session-cookie mode is read-only. Set an MCP token for write access (create, select, approve)."

Requests with neither credential are rejected with HTTP 401 before the MCP server runs.

## The human-in-the-loop contract

The user's name goes on every post. These rules override convenience:

1. **Never approve or publish without an explicit instruction in the current conversation.** "Generate a post for review" authorizes drafting only.
2. **Show, then ask.** When a run reaches `awaiting_approval`, present the draft and its source before suggesting approval.
3. **The default run pauses twice.** `create_run` sends `wizard: true`, which forces review at article selection and at draft approval even for tenants with auto-approve enabled. Pass `force_review: false` only when the user explicitly opts into unattended publishing.
4. **Record the user's choice.** `select_source` captures the user's article pick, or their explicit "pick the best one".
5. **Credits:** a post credit is spent only when the post actually publishes — for a manual run, moments after `approve_draft`; a failed publish spends nothing. Say so before asking for approval.
6. **Report only what the pipeline confirmed.** `approve_draft` returning `{ok: true}` records the approval — not the publication. Wait for the run's terminal status and report that: the LinkedIn URL on success, the exact error on failure. Never tell the user a post is live based on the approval reply alone.
7. **Expertise mode still ends with the user.** The brief is the user's own position, so every answer must be their words — never invent or "improve" one. Source search is a choice, not an auto-pick: present the candidates and record the user's pick; choose for them only on an explicit "pick the best one". An expertise run always pauses for approval, whatever the tenant setting or token scope.

## Voice setup: train the writing voice

When the user asks PostForge to learn how they write ("train my voice on these posts", "set up my writing voice"), work through this ladder. It needs a `draft-only` token or better; nothing here publishes.

1. **Discover.** `list_voice_profiles()`. If a sample-capable profile already exists (`source_type` is not `"preset"`), reuse it; otherwise `create_voice_profile(name=...)`. `create_voice_profile` also accepts `editorial_preferences` (see step 4) when the user has already stated formatting preferences. Never add samples to a preset profile — it is refused ("Preset profiles do not accept samples. Create a custom profile instead."). If the user only wants a known style, `create_voice_profile(source_type="preset", preset_key=...)` activates immediately and skips the rest of this section.
2. **Collect 3–10 samples.** `add_writing_sample(profile_id, text)` is text-only; get the text one of three ways, in this order:
   - **Paste** — the user pastes a post; pass it through.
   - **Workspace file** — the user names a file; read it with your file tools and pass the text. (The MCP server runs inside the PostForge API container and cannot reach the user's disk.)
   - **URL** — fetch the page with **your own** web tool and pass the text. PostForge makes no outbound request for a user-supplied URL.
   Truncate to 8,000 characters per sample; set `sample_kind` to `article` for articles and `linkedin_post` for posts. Aim for 3–10 samples, ideally all in the user's own voice. On "An identical sample already exists." report it and move on — do not retry the same text.
3. **Analyze and review.** Once at least 3 samples are active, `analyze_voice(profile_id)` — it blocks on one LLM round-trip and is rate-limited (6 per hour per tenant). Present the returned `style_guide` and the profile's sample previews (`get_voice_profile`) to the user, ask whether the result sounds like them, and refine before activation. This is the agent-side equivalent of the web's guided voice review; do not activate a profile the user has not seen.
4. **Refine and manage (optional).** `update_voice_profile(profile_id, ...)` changes the guide, instructions, name, or editorial preferences — patch semantics: pass only the fields to change. Edits shape every future draft immediately; re-running analysis supersedes guide edits. Editorial preferences are user choices, not model inferences, and survive re-analysis:
   ```json
   {
     "allow_parentheses": false,
     "allow_dashes": false,
     "hashtag_mode": "required",
     "paragraph_style": "flowing",
     "structure_mode": "guided",
     "closing_style": "either"
   }
   ```
   `hashtag_mode` is `required` / `optional` / `none`; `paragraph_style` is `flowing` / `line_breaks`; `structure_mode` is `guided` / `free`; `closing_style` is `either` / `question` / `statement`. Omitted keys take their defaults. These preferences change punctuation, hashtags, layout, and structure only — plain text, the source URL, the length limit, and the sensitive-topic policy stay fixed. Phrasing that reads as machine-written is flagged as advice, never a rejection, so a precise technical term the author would use stays in the draft. Manage samples the same way: `update_writing_sample` pins one (pinned samples are preferred as drafting examples), deactivates one to keep it saved but out of play, or relabels it; `delete_writing_sample` removes it outright. There is no stored sample order — pinning plus the profile's `example_selection_strategy` is the ordering mechanism. Voice & brand in the web app is the alternative surface.
5. **Activate.** `activate_voice(profile_id)`, only on an explicit instruction. State the consequences before calling: every future draft uses this voice and its editorial preferences, and the previously active profile is archived (reversible by activating another).

## Choose the starting point

Ask once, before shaping anything: **From an article** (the default) or **From your expertise**. Do not ask when the user already said which — "draft from this report" is article mode; "I want to write about what I keep seeing in audits" is expertise mode.

Expertise mode is for a position the user already holds: a pattern, a decision rule, a belief they disagree with, a question clients keep asking, or a framework. The article attached afterwards is evidence, not the subject — the brief leads the draft.

Work through it in this order:

1. `get_draft_options()` — the active voice and its editorial preferences, plus `expertise_brief`: five kinds, each with its two kind-specific fields, and the three common fields (`reader`, `boundary`, `takeaway`), each with its question and `max_chars`.
2. Offer the five kinds with their labels and descriptions. On the pick, ask all five questions **in one grouped message**, with an example answer for each. Record the user's words verbatim — never paraphrase an answer into something more publishable, and never fill one in. The server enforces `max_chars`.
3. Summarize the brief back as five labelled answers and get a yes before searching. This is the last cheap place to fix a misunderstanding.
4. Source: paste a URL, or call `discover_sources_for_brief(expertise_brief)` and present the candidates with their `relationship` (`supports` / `challenges` / `related`), `reason`, and `publish_date`. Only individual verified articles come back, so a short list means little citable was found — say that rather than padding it. The user picks; never auto-pick. If the search fails or returns nothing, say so and ask for a URL — the paste path always works.
5. `create_run(article_url=..., expertise_brief={"kind": ..., "fields": {...}}, generate_image=...)`. Never pass `perspective`: the brief kind is the structural authority, and the server refuses a take alongside a brief.
6. Poll `get_run(run_id)` until `awaiting_approval` — there is no article-selection pause, because the source is already attached — then run **The approval checkpoint**.

There is no saved-take menu and no suggested interpretation in this mode. The voice still applies: it controls surface formatting (punctuation, hashtags, layout) while the brief controls the position and the shape.

## Workflow: draft from an article

1. This is the **From an article** path. Confirm the tenant has topics configured; if not, ask for an article URL instead.
2. **Shape the draft first.** Call `get_draft_options()` and settle two questions with the user before starting the run — ask them together in one message. Skip any question the user already answered, and skip both only when the user explicitly waives them ("just draft it").
   - **Voice** — drafts always use the tenant's *active* voice profile; there is no per-run choice, so confirm rather than offer. Its `editorial_preferences` are applied automatically; to change formatting, update the profile. If several profiles exist, name the active one and ask whether it is the right voice for this post (to switch, `activate_voice` another profile — see Voice setup — or change it in Voice & brand). When none is configured, say drafts will use the default settings-based voice.
   - **Their take** — the user's own perspective is what makes the post worth reading. When the source article is already known (URL runs), lead with fresh article-derived angles: `suggest_takes(article_url)` returns three from the article itself — offer those first, then `default_take` / a saved take from `saved_takes[]` verbatim, or the user's own words. Pass the result as `create_run(perspective=...)`. For discovery runs the article is not chosen yet: settle the take with the saved options up front, and after the candidate pick (step 5) you may refresh with `suggest_takes(<candidate url>)` and pass the updated take to `select_source(perspective=...)`. Omit the take only if the user has nothing to add.
3. `create_run()` for topic discovery, or `create_run(article_url=...)` to draft from a specific source — carrying the take from step 2 (`perspective`). Pass `generate_image: true` only when asked (paid feature).
4. Poll `get_run(run_id)` until status is `awaiting_article_selection`, then `list_candidates(run_id)`. Offer the selectable ones with their publication date; a row carrying `rejected: true` with an eligibility or freshness reason is a page the search refused (a section, listing, or out-of-window date), so name it as unavailable in plain words instead of offering it.
5. `select_source(run_id, candidate_id)` records the choice and starts drafting.
6. Poll `get_run(run_id)` until `awaiting_approval`, then run **The approval checkpoint** below before anything else.

**Blocked by a 409?** `create_run` refuses while the tenant has an active run. Call `get_run()` with no arguments to see it (id, status, draft if parked). Then either wait for it, resume it (step 4 onward), or — only with the user's agreement for a stale run — `cancel_run(run_id)` and retry. Never ask the user for a session cookie or a run id; the token covers all reads.

## The approval checkpoint

When a run is parked at `awaiting_approval`, the user's name is about to go on a public post — this is the step that matters most. Work through it in order:

1. **Present.** The source (title + link), the full draft in a quoted block, and — when `get_run` carries validation notes — a short "automated checks" line, labelled honestly (formatting and quality checks, not fact-checking). A claim check, when run, is labelled separately as a comparison with the selected source, not as fact-checking. With file tools, also write the draft to `postforge-draft-<run_id>.md` in the workspace and give the user the path — the editable copy is part of the presentation, not an option to request later.
   When `get_run` carries `expertise_brief`, put a **Your brief** block above the draft: the kind and the five labelled answers, then the active voice and its editorial preferences. Say plainly that the attached source is evidence, not the subject. The brief is read-only once the run starts — to change it, the user edits the draft or cancels and starts again.
2. **Offer the review options once.** Approve as-is, edit, or run a claim check first (paid plans; the verdicts are stored on the draft) — where editing means: change `postforge-draft-<run_id>.md` and say when it is ready, or describe the changes in chat. Always name the file path in the offer when you wrote one. Do not re-offer on every turn.
3. **Take edits.** When the user says the file is ready, read it back and apply the text. Chat edits: apply as described and re-present the full text. Where the client supports a pop-up form, `edit_draft(run_id)` offers it — today's hosted transport answers `{"supported": false}` immediately, so never wait on it; a `{"cancelled": true}` result means the user backed out and nothing changed. Re-write `postforge-draft-<run_id>.md` whenever the draft text changes so the file always matches the current draft, and delete it once the run is approved or abandoned.
4. **Image step — before any approval.** Mirror the web's review controls. Never generate without explicit opt-in (it is paid spend).
   - If `get_run` shows `generate_image: true`, an image is already generated (or still generating): call `get_image` to present it — on the "still generating" error, wait a few seconds and retry; generation usually completes within a minute. Offer the web's controls: "Generate alternate (N left)" via `generate_image(run_id)` (the reply's `remaining` is N — when regenerating, poll `get_image` again), or remove it (text-only).
   - If no image was requested, **ask explicitly**: "Want an AI-generated image to accompany this post?" — it requires an active subscription. On opt-in: `generate_image(run_id)`, then poll `get_image` until it returns the picture, and present it with the same regenerate/remove controls. On a refusal like "Image generation is available on paid plans.", say so plainly and continue text-only.
   - **Make sure the user can actually see it.** The `get_image` picture renders inline where the harness supports image content; when it does not (or the user asks for it), give them the run's review page in the web app — `<MCP-origin>/runs/<run_id>`, using the same origin as the MCP server URL from Setup, opened where they are signed in. It shows the image at full size with the same regenerate/remove controls.
   - The image decision sets the approval: kept → `include_image=True`; removed or declined → `include_image=False`. Never silently publish (or drop) an image the user has not seen.
4b. **Run the claim check if the user chose it.** After edits and the image decision, call `check_claims(run_id, draft_body=<final text>)`. Present the summary and any flagged claims with their source passages. Say it compares the draft to the selected source and does not verify the article; the verdicts are stored with the text they were computed from, and a claim the source contradicts can hold the send back until the author fixes it or records why they are keeping it. Only on the user's explicit decision to keep such a claim, send them to the run's review page in the web app (`<MCP-origin>/runs/<run_id>`) to acknowledge it — the acknowledgement is a named, reasoned human action and is not offered as a tool call here. On "Claim checks are available on paid plans.", say so and continue without it.
   For an expertise run, present the two groups separately: **Compared with the source** first, then **From your brief, not checked against the source**. A `from_brief` claim is the author's own practice observation — say that, and never call it unsupported or unverified. When `basis_available` is false, say the split could not be made this time and every claim was compared with the source.
5. **Show the final text verbatim** — the full post as it will be published (not just a diff) plus a one-line summary of what changed, and the image decision.
6. **Approve only on an explicit instruction.** Two publishing outcomes, plus an optional voice-learning flag:
   - **Publish now** — `approve_draft(run_id, edited_body=<final text if changed>, include_image=<per step 4>)`. Say what that does: for a manual run it starts publishing to LinkedIn immediately (the credit is spent when the publish succeeds); undo means deleting the post on LinkedIn.
   - **Schedule for a later time** — when the user asks for a future date/time, call `approve_draft(..., publish_at=<ISO 8601 with offset>)`. Ask for the date, time, and timezone if not given, convert, and read the resulting local time back before calling. The reply `{"publication": "scheduled", "scheduled_publish_at": ...}` is final — nothing publishes now; the credit is spent at publish, and the post stays cancellable until then. To change the time, approve again with a new `publish_at`; to publish immediately, approve without it.
   - **Voice learning (optional)** — when the user explicitly says the draft sounds like their writing, add `save_as_sample=true` to `approve_draft`. It attaches the final text to their active custom voice; unchanged AI drafts are never learned without it. It requires the tenant's voice-learning opt-in and has no effect on preset profiles. If nothing is learned, point the user to Settings → Voice to enable it.
7. **Verify the publish — the approval reply is not proof.** A refusal can come back immediately: when `approve_draft` errors with a rule's sentence ("This draft is 3001 characters, 1 over LinkedIn's 3000-character limit", "N claim(s) in this draft contradict the source", "The draft changed after the last claim check"), nothing was approved and the run is still parked at `awaiting_approval`. Report the sentence and act on it — shorten the text, re-run `check_claims` after an edit, or send the user to the review page to acknowledge a claim — then approve again on the user's instruction. `{ok: true}` only records the approval; the pipeline publishes in the background right after it. Unless the reply says `"publication": "scheduled"` (a windowed occurrence or a one-shot scheduled post queued for later — confirm the `scheduled_publish_at` time instead of polling; nothing is publishing now, and the outcome lands on the run when its target fires), poll `get_run(run_id)` every few seconds until the status is terminal. The run may briefly still read `awaiting_approval`, then `running` — keep polling until `succeeded`, `failed`, or `cancelled` (usually under a minute; an image upload adds time). **If it is still `awaiting_approval` with a `validation_issues` finding after you approved, the send was refused, not delayed**: say so, quote the finding's explanation, and stop polling. Report the terminal outcome honestly:
   - **`succeeded`** — present the LinkedIn post URL from that same payload (`post_url`) as the proof of publication.
   - **`failed`** — say plainly that the post did **not** publish, and why: quote the `error` text verbatim (e.g. `Duplicate detected: <url>`) with its `error_kind`. Then give the honest way forward — the run is closed (a finished run cannot be cancelled and does not need to be; it never blocks a new one), so the retry is a fresh `create_run`: a different source or angle, or a text-only run (skip the image at the image step) when the `error` points at the image. For a duplicate refusal specifically: that source URL already has a published post — offer a different source, or, only on the user's explicit agreement to republish it, a new `create_run(..., allow_duplicate=true)`.
   - **`cancelled`** — nothing was published; say so plainly. (Rare after approval.)
8. **A decline changes nothing** — the run stays parked; the web UI is the alternative surface. Skip the edit offer (read-only preview) only when the user explicitly asked to just see the draft — and even then, show the full text before any approval.

## Error handling

Tools raise an error carrying the API's own detail string. Branch on it:

| Detail contains | Meaning | Do |
|---|---|---|
| `already in progress` | 409: one active run per tenant | `get_run()` (no arguments) shows it — wait, resume, or cancel if stale |
| `Add a topic or paste an article link` | 422: no topics configured | Ask for topics or a URL |
| `Attach a source before drafting` | 422: an expertise run without an article URL | Ask for a URL, or run `discover_sources_for_brief`, then retry |
| `Use the expertise brief or a take, not both` | 422: article-first and expertise inputs mixed | Drop `perspective` — the brief carries the shape |
| `Choose an insight kind` / `Add your expertise brief` / `Add <field>` / `Shorten <field>` | 422: the brief failed validation | Quote the message, ask the user for that one answer, retry — never fill it in for them |
| `Source search failed. Please try again.` | 502: the brief-driven search failed | Say search is best-effort; ask for a URL and continue with `create_run` |
| `That source is not a verified article` / `outside the freshness window` / `section or listing, not an article` | 409: the picked candidate is a page the article policy refuses | Offer another candidate or a rediscover; never retry the same id |
| `No verified articles found` | the search returned only listing or undated pages | Say nothing citable was found; offer a pasted URL, a topic tweak, or a broader lookback window |
| `Trial posts used up` | 402: credit cap reached | Tell the user; do not retry |
| `not awaiting article selection` | 409: wrong state (run moved on) | Poll again |
| `already published a post from this link` | 422: source reused within the 60-day window | Tell the user the date; ask whether to republish; only after their explicit agreement retry `create_run(..., allow_duplicate=true)` |
| `publish_at must include a timezone offset` / `Choose a publish time at least 5 minutes in the future` | 422: the scheduled time was naive or too soon | Convert the user's local time to ISO with an offset, at least 5 minutes ahead; retry once |
| `Scheduling is part of a paid plan` | 403: enabling a recurring schedule without a paid plan | Tell the user scheduling is a subscriber feature; offer manual publishing |
| `Update your app to edit this publishing-window schedule` | 409: a windowed schedule cannot be partially rewritten | Send the complete windowed payload: `days_of_week` plus `end_hour` and `end_minute` |
| `Run cannot be cancelled` | cancel attempted on a finished run | Nothing to cancel — succeeded/failed runs never block a new one; start a new run (removing history is a web-app action) |
| `Session-cookie mode is read-only` | connected with a cookie, not a token | Mint a token; writes need bearer auth |
| `Claim checks are available on paid plans` | 403: trial plan | Tell the user; continue without the check |
| `not awaiting approval` | 409: wrong state | Poll `get_run` and continue from the current state |
| `Could not load the source article` / `Claim check unavailable` / `Could not check the draft` | 502: source or provider failure | Tell the user; retry once, then continue to approval without blocking |

No or invalid credentials fail the connection itself with HTTP 401 — check the `Authorization` header in the MCP config.

## Tools used by these workflows

| Need | Tool |
|---|---|
| Shaping context and the brief registry | `get_draft_options()` |
| Start a run, article or expertise | `create_run(...)` |
| Find sources for a brief | `discover_sources_for_brief(expertise_brief)` |
| Record the article pick | `select_source(run_id, candidate_id)` |
| Open a paused run's candidates | `list_candidates(run_id)` |
| Refresh article-derived angles | `suggest_takes(article_url)` |
| Read or poll a run | `get_run(run_id?)` |
| Compare claims with the source | `check_claims(run_id, draft_body?)` |
| Edit in a pop-up form | `edit_draft(run_id)` |
| Approve, publish, or schedule | `approve_draft(...)` |
| Image opt-in and review | `generate_image(run_id)`, `get_image(run_id)` |
| Abandon a stale run | `cancel_run(run_id)` |
| Voice setup and management | `list_voice_profiles`, `get_voice_profile`, `create_voice_profile`, `update_voice_profile`, `add_writing_sample`, `update_writing_sample`, `analyze_voice`, `activate_voice`, `delete_writing_sample`, `delete_voice_profile` |
| Recurring publishing window | `get_schedule()`, `set_schedule(...)` |

Full contracts, scopes, and HTTP mappings for all 25 tools:
[`references/tools.md`](references/tools.md).

Scopes in one line: reads work with any token; drafting, claim checks, brief
search, image generation, and voice writes need `draft-only` or `full`;
`approve_draft` and `set_schedule` need `full`.

## Scheduling, honestly

Two mechanisms exist:

- **One post, one time** — `approve_draft(publish_at=...)` schedules the current draft for a specific instant (web: "Schedule for later" in the review panel). A queued post does not block a new run, so several may be queued; each publishes at its chosen minute (or on the next scheduler tick after downtime) and is cancellable until its send starts.
- **A recurring publishing window** — a tenant-level daily/weekdays/weekly window, a paid feature, configured through `set_schedule` or Configure → Automation and read with `get_schedule`. Windowed drafts are prepared ahead and queue for the window's target. Enabling a schedule on an auto-approve tenant publishes without review — confirm with the user first.

There is no bulk or per-post calendar editing over MCP; manage individual queued posts through their runs, and the recurring window through `get_schedule`/`set_schedule` or the web.
