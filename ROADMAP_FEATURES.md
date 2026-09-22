# Road map features

Status: proposed. This document plans future work; it does not provision cloud services or implement any of these features.

## Product goal

Turn the UGC Notebook into a practical campaign workspace. A founder should be able to sign in, create a project for an app or startup, collect relevant UGC references, plan four weeks of experiments, and use recorded results to decide what to repeat, revise, or stop.

The core workflow is:

**Audience hypothesis → reference → content idea → experiment → published post → results and comments → weekly decision → next experiment.**

Every experiment should retain this chain of context. The goal is better campaign decisions, not merely storing links or collecting view counts.

## Starting point and scope

The existing website is a public, self-contained notebook hosted on GitHub Pages. Its six mind maps, source links, and expanded notes remain useful learning material.

The proposed product adds a separate authenticated dashboard backed by Google Cloud and Firebase. In this document, “admin dashboard” means a user managing their own campaign projects. Signing up does not grant access to other users' projects or make someone a platform administrator.

The first usable release includes infrastructure, authentication, project management, a reference library, a four-week planner, experiment records, and weekly reviews. Creator operations follow as the next milestone. Automated social publishing, payments, and AI-generated videos are outside the first release.

The four-week schedule below is a **campaign template for users**, not a promise that all engineering milestones will be delivered in four weeks. Engineering dates and owners should be assigned when each milestone is scheduled.

## Milestone 1 — Google Cloud foundation and Docker

**Outcome:** developers can run the application locally and deploy a reproducible version to a staging environment.

**Dependencies:** none. Confirm the Google Cloud project, billing owner, deployment region, and spending budget before provisioning.

- [ ] Create separate staging and production Google Cloud/Firebase projects with documented configuration.
- [ ] Package the dashboard web application and its API in a Docker image. Use a multi-stage build, a non-root runtime, a health endpoint, and the port supplied by the hosting environment.
- [ ] Provide a local Docker Compose workflow for the application and Firebase emulators, plus an example configuration file without credentials.
- [ ] Push versioned images to Artifact Registry and deploy the application to Cloud Run. Serve the dashboard and API from the same origin initially.
- [ ] Add GitHub Actions checks for pull requests and a deployment workflow for the application release branch. Authenticate CI through Workload Identity Federation rather than committed service-account keys.
- [ ] Configure runtime service accounts, Secret Manager, structured logs, error monitoring, budget alerts, and initial scaling limits. Document that budget alerts alone do not impose a hard spending cap.
- [ ] Document how to deploy a specific image and restore the previous working Cloud Run revision.
- [ ] Keep the current GitHub Pages notebook available. Select the dashboard's public URL explicitly; do not assume it should inherit the account's existing custom domain.

**Done when:** a clean checkout runs using the documented local workflow; CI deploys the same image to staging; the health check succeeds; an intentional staging rollback succeeds; no runtime secret appears in the repository or image.

Implementation references: [Cloud Run container deployment](https://docs.cloud.google.com/run/docs/deploying), [CI authentication with Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-deployment-pipelines), and [Cloud Run secrets](https://docs.cloud.google.com/run/docs/configuring/services/secrets).

## Milestone 2 — Firebase database and user authentication

**Outcome:** users can sign in and access durable, private project records across devices.

**Dependencies:** Milestone 1's local and staging environments.

- [ ] Use **Cloud Firestore** as the Firebase database for users, projects, references, campaign plans, experiments, posts, observations, and reviews.
- [ ] Integrate Firebase Authentication with Google sign-in and email/password sign-up, email verification, password recovery, sign-out, and clear expired-session handling.
- [ ] Keep passwords in Firebase Authentication; never store them in project documents.
- [ ] Route private application data through the Cloud Run API. Verify Firebase ID tokens on protected requests, derive the user ID from the verified token, and check access to the requested project on every operation.
- [ ] Make projects private to their owner initially. Retain a membership model for later owner/editor/viewer collaboration, without enabling broad sharing in the first release.
- [ ] Deny direct browser access to private Firestore documents in Security Rules for this API-only design. Use narrowly scoped service-account IAM and explicit API authorization for server access.
- [ ] Validate record shapes, allowed values, cross-record project membership, and required fields on the server. Do not trust a user ID or owner ID supplied by the browser.
- [ ] Add versioned database indexes, bounded queries, pagination, and a consistent approach to server timestamps and concurrent edits.
- [ ] Use Cloud Storage for Firebase for permitted image attachments; keep file metadata in Firestore. Authorize upload/download operations against the project, validate file type and size, and avoid public bucket access.
- [ ] Define export, archival, deletion, and backup/restore behavior. Deleting an account or project must also handle its subcollections and attachments, rather than only deleting a parent document.

**Done when:** two test users can create and reopen their own records, but neither can read, modify, attach files to, or export the other's project by changing a URL or ID. Unauthenticated and expired-token requests fail safely. Data survives sign-out and sign-in on another device. A staging restore has been demonstrated.

The server's Firebase Admin SDK bypasses Firestore Security Rules, so API authorization tests are required in addition to rules tests. Use the [Firebase Emulator Suite](https://firebase.google.com/docs/emulator-suite), follow [Firebase ID-token verification](https://firebase.google.com/docs/auth/admin/verify-id-tokens), and test the default-deny rules using [Firebase's rules testing guidance](https://firebase.google.com/docs/firestore/security/test-rules-emulator).

## Milestone 3 — Project dashboard and audience brief

**Outcome:** a user can create a campaign project with enough context to make relevant content decisions.

**Dependencies:** Milestone 2.

- [ ] Build a project list with create, open, edit, archive, and duplicate-template actions. Duplication copies selected planning material, not previous campaign performance.
- [ ] Make the dashboard open on the current project and campaign week, showing the next action, scheduled work, missing observations, unanswered product questions, and the next review.
- [ ] Add a guided project brief based on the notebook's [audience chapter](index.html#audience).
- [ ] Provide a project switcher and useful empty states that lead to creating a brief, saving a reference, or starting a campaign.
- [ ] Link dashboard tasks back to the relevant notebook chapter and source notes. A user should be able to move from advice to a concrete task without losing their place.

### Project fields

- **Identity:** project name, app/startup name, description, website or store URLs, product category, niche, and project status.
- **Audience:** persona, age range when relevant, language, geography, situation, immediate goal, pain points, objections, and communities or creators they follow.
- **Value proposition:** the specific problem, visible product payoff, supporting evidence, and what the product can honestly demonstrate.
- **Campaign setup:** primary platform, optional secondary platforms, start date, timezone, available production capacity, cadence hypothesis, and budget/currency when applicable.
- **Goals:** primary learning question, chosen outcome metric, baseline if known, target, and measurement method. A download target is optional and is never a guaranteed result.
- **Voice:** familiar phrases, slang/emojis used by the niche, tone examples, and language or claims to avoid.
- **Ownership:** owner, creation/update timestamps, and future membership references.

Require a project name, niche, audience hypothesis, and problem/payoff to begin a campaign. Allow the remaining brief to evolve as the user learns.

**Done when:** a user can create two distinct projects, switch between them, update their briefs, and archive one without mixing references or results. The dashboard identifies the next useful action for both an empty and an active project.

## Milestone 4 — UGC reference library and content ideation

**Outcome:** users can save niche-relevant examples and turn observations into their own testable ideas.

**Dependencies:** Milestone 3.

- [ ] Support manual reference entry from TikTok, Instagram, YouTube, or another source URL, as well as written examples and user-supplied screenshots.
- [ ] Support editing, tagging, filtering, pinning, and marking references as unavailable or no longer relevant. Detect duplicate source URLs within a project.
- [ ] Organize references by niche/sub-niche, audience, problem, platform, content format, hook type, and research status.
- [ ] Let users write original sample hooks, scripts, or storyboards alongside saved references. Keep original ideas distinguishable from external examples.
- [ ] Add “Create experiment from this reference” and allow an experiment to link to multiple references.
- [ ] Separate observed facts from interpretations: a high view count is observable; “this hook caused conversions” requires supporting evidence.
- [ ] Start with links and manual observations. External embeds may be used where supported, but a blocked embed or removed video must not erase the user's notes.

### Reference fields

- **Source:** title, canonical URL, platform, creator/account, published date if known, saved date, and optional screenshot or relevant timestamp.
- **Relevance:** niche, intended persona, viewer situation, pain point, language, and a written reason it applies to this project.
- **Creative breakdown:** exact hook excerpt or paraphrase, format, video duration, story beats, demonstration/payoff, call to action, and tone.
- **Comment evidence:** selected product questions, recurring objections, useful audience phrasing, and a link or timestamp identifying the observation.
- **Observed performance:** views and available engagement counts, when they were observed, and their source. Missing metrics remain unknown rather than becoming zero.
- **Interpretation:** why the example may work, confidence/evidence notes, what to adapt, and what remains untested.
- **Workflow:** tags, research status, linked ideas/experiments, contributor, and updated timestamp.

### Notebook-inspired idea templates

1. **Hook and demo:** audience problem → personal opening → product action → visible payoff.
2. **Conversational text:** relatable observation/question → readable overlay → simple supporting clip → discussion prompt.
3. **Storytelling:** situation → friction → discovery → realistic outcome.

Allow users to compare three hook variants while keeping the demonstration consistent. Keep the person's real voice and the app's actual capabilities central to the brief.

**Done when:** a user can save a relevant example, explain why it matters, retrieve it using niche/format filters, create an original idea, and link that idea to a scheduled experiment. Source notes remain available even if the original video becomes inaccessible.

## Milestone 5 — Four-week campaign planner and production records

**Outcome:** notebook recommendations become dated tasks and traceable experiments.

**Dependencies:** Milestones 3 and 4.

- [ ] Create a 28-day campaign from a project brief, using the four-week template below. Allow optional days 29–30 for wrap-up to align with the notebook's original 30-day framing.
- [ ] Generate editable weekly goals and daily tasks relative to the campaign start date and timezone. Shifting dates should preserve actual publication times and historical observations.
- [ ] Provide calendar and weekly-board views with planned, in progress, published, reviewed, skipped, and blocked states where relevant.
- [ ] Let users choose their production capacity. The source's 2–3 posts per day and two-hour spacing are optional experiments, not required defaults or platform guarantees.
- [ ] Link each content experiment to its audience, problem, reference material, hypothesis, format, hook, and expected signal.
- [ ] Track scripts, creator/owner, production status, planned publication, actual publication URL/time, and follow-up observation tasks.
- [ ] Keep platform publications separate from the underlying creative so a cross-post does not merge unrelated platform results.
- [ ] Record what changed between variants and preserve the original hypothesis after results arrive.

### Four-week campaign template

#### Week 1 — Understand the audience and launch first tests

**Days 1–3:** write the audience brief, inspect niche feeds and comments, build the language cheat sheet, and collect relevant references. Select three problems and turn them into hooks.

**Days 4–7:** make founder-led content using the notebook's three formats at a sustainable pace. Record each post and reply to genuine questions.

**Weekly review:** which problem did people recognize, and what did they ask next?

**Required records:** audience hypothesis, annotated references, first content hypotheses, publication records, and initial comment observations. Reference/post count targets are editable rather than forced quotas.

#### Week 2 — Compare messages and strengthen intent signals

**Days 8–14:** vary hooks while keeping the core demo or payoff consistent where possible. Continue collecting niche language and objections. Record metrics at comparable observation windows, such as 24 hours, 72 hours, and seven days, when available.

**Weekly review:** which audience/problem/format combination generated useful product questions? What should change in the next brief?

**Required records:** linked variants, timestamped metric observations, categorized questions, reply/follow-up status, and a repeat/revise/stop decision with evidence.

#### Week 3 — Repeat promising angles and prepare delegation

**Days 15–21:** inspect posts that outperform the campaign's baseline and create follow-up variations. Treat the source's 10k-view marker as an editable discovery filter. Check downstream visits, installs, or activation only where attribution is available.

If the message is becoming repeatable, prepare a creator brief and optionally run a week-long paid trial. Hiring is not required to complete the campaign.

**Weekly review:** did the promising angle work again, and can another person understand the brief?

**Required records:** a winning-angle hypothesis, a reference-backed brief, follow-up experiments, attribution notes, and optional creator trial records.

#### Week 4 — Audit, document, and plan the next cycle

**Days 22–28:** compare mature results, review creator work if applicable, document repeatable formats, and decide which experiments to continue, change, or stop.

**Weekly review:** what did the campaign teach us, and what evidence supports the next decision?

**Required records:** campaign review, reusable playbook entries, unresolved audience/product questions, and a draft next-cycle plan. Optional days 29–30 close out observations that are still maturing.

**Done when:** a user can launch a dated four-week plan, reschedule a task, publish and link an experiment, complete each weekly review, and start the next cycle without overwriting the previous campaign's history.

## Milestone 6 — Measurement, comments, and weekly decision dashboard

**Outcome:** the dashboard helps users distinguish reach from intent and product outcomes.

**Dependencies:** Milestone 5. Together, Milestones 1–6 define the first end-to-end release.

- [ ] Allow manual metric entry first, with optional structured CSV import and a preview that reports invalid or duplicate rows.
- [ ] Store metrics as timestamped observations per published post: views, likes, total comments, shares, saves, available profile/website visits, attributable installs, and activation events where known.
- [ ] Record the source, observation window, timezone, attribution method, and availability of each metric. Preserve missing values as unknown.
- [ ] Add a comment log with source link, excerpt/summary, category, assigned owner, reply status, follow-up action, and related product insight.
- [ ] Use comment categories such as app discovery, feature question, use-case fit, objection, onboarding friction, and unrelated engagement.
- [ ] Summarize reach, product-interest signals, downstream outcomes, production effort, and costs separately. Avoid a single opaque “viral score.”
- [ ] Compare experiments by audience, pain point, hook, format, creator, platform, and observation age. Show sample size and incomplete data alongside rankings.
- [ ] Add a weekly review form: strongest evidence, counter-evidence, objections, product feedback, repeat/revise/stop decision, next hypothesis, and linked supporting records.
- [ ] Let users promote a reviewed result into a project playbook with its context and limitations, then seed the next campaign from it.
- [ ] Export project references, campaign records, and reviews to portable formats, preserving IDs and links between records.

### Measurement rules

- Do not add a post's cumulative view snapshots together. Compare the latest comparable snapshot or calculate a clearly labeled interval change.
- Deduplicate imported observations using the post, metric source, and observation timestamp or source record ID. Keep a correction history for manually edited observations.
- A signal-comment rate can be displayed as qualifying comments divided by views, but it is a comment-event ratio, not a unique-person conversion rate.
- Only show an install conversion rate when installs and the denominator use a compatible attribution window and definition. Otherwise show separate counts with an “attribution unavailable” explanation.
- Calculate cost per attributed install only when cost and install attribution cover the same scope. Handle unknown or zero denominators explicitly.
- Do not present a target of 1,000 downloads, a 10k-view threshold, account warm-up, or pre-post engagement as a proven platform rule or guaranteed outcome.

**Done when:** a user can trace a weekly decision from its reference through the content variant, published post, observations, and comment evidence. Reimporting the same data does not inflate totals, and missing attribution never produces a fabricated conversion result.

## Milestone 7 — Creator operations and collaboration

**Outcome:** a project with a repeatable message can coordinate creators without losing the learning loop.

**Dependencies:** Milestones 4–6. Deliver after the first end-to-end release; founder-only campaigns must remain fully usable.

- [ ] Store creator profiles, niche/platform fit, portfolio links, availability, communication notes, and trial status.
- [ ] Track a week-long paid trial with agreed deliverables and a rubric covering clarity, lighting/audio, audience fit, communication, and response to feedback.
- [ ] Build creator briefs from the project persona, relevant reference examples, authentic voice guidance, proven message, and the next learning objective.
- [ ] Add draft review, specific feedback, revision history, approval status, and handoff checklists.
- [ ] Record flat-fee and CPM bonus terms: currency, eligible posts/views, rate per 1,000 views, measurement window, cap if any, and an estimated payable amount. Do not initiate payments.
- [ ] Review creative quality, reliability, intent, and available product outcomes alongside reach. A CPM incentive must not become the only ranking criterion.
- [ ] Add owner/editor/viewer memberships with revocable invitations and explicit permissions. Only owners manage membership and destructive project settings.
- [ ] Record membership changes, approvals, and significant edits in a project activity history. Creator records do not automatically grant the person dashboard access.

**Done when:** an owner can run and review a paid trial, give actionable feedback, compare agreed metrics, and document a retain/coach/stop decision. Removing a collaborator immediately prevents further project API access.

## Data model outline

Use project-scoped Firestore collections rather than growing arrays inside one large project document. Exact collection names and indexes should be finalized during Milestone 2.

- **Users:** Firebase user ID, display preferences, timezone, onboarding state, and account timestamps.
- **Projects and memberships:** owner, audience/product brief, status, and user-to-project roles. Initially only the owner membership is active.
- **References:** source information, annotations, niche/format tags, relevance notes, and attachment metadata.
- **Ideas and experiments:** original creative, linked references, hypothesis, variable under test, variants, and production state.
- **Campaigns and tasks:** project, start date, timezone, template version, weekly goals, daily actions, due dates, status, and actual completion times.
- **Posts:** experiment/variant, platform, creator, publication URL, and actual publication time.
- **Metric observations:** post, observation timestamp/window, metric values, provenance, attribution notes, and import/correction identifiers.
- **Comment observations:** post, source, excerpt/summary, category, reply state, and related insight.
- **Weekly reviews and playbook entries:** campaign/week, decision, rationale, evidence links, reusable lesson, and next action.
- **Creators, trials, and briefs:** project-specific working relationship, agreed deliverables, feedback, and optional compensation terms.
- **Activity records:** actor, project, action, affected record, and timestamp for significant changes.

All project records carry a project identifier, creation/update timestamps, and creator/editor identity. The API validates that linked records belong to the same accessible project. Attachments are stored outside Firestore with project-scoped ownership metadata.

Support queries for a user's project list, project references by tag/format, campaign tasks by date/status, and observations by post/time. Use pagination from the start; keep derived dashboard summaries rebuildable from source records.

## Example end-to-end journey

1. A founder creates a language-learning app project for people who freeze during real conversations.
2. They save a relatable UGC example, identify the hook, annotate relevant comments, and explain why the niche matches.
3. They create three original hook variants using the same honest product demonstration and place them in week one's plan.
4. Each published post receives its own URL, observation times, metrics, and product-question log.
5. At the weekly review, they compare similarly aged posts and decide to repeat one angle based on both audience questions and available outcome data.
6. They create a follow-up experiment linked to that decision. Once the format is repeatable, they generate a creator brief from the same evidence.
7. At day 28, the dashboard produces a review and next-cycle draft while preserving all original references and observations.

## Release checks and success measures

Before the first production dashboard release, complete a staging walkthrough of the journey above using two isolated user accounts and a campaign with mixed complete/missing metrics.

- [ ] Verify authentication, per-project authorization, attachment access, and export boundaries through API and emulator tests.
- [ ] Verify duplicate imports, timezone/date changes, metric corrections, and unknown denominators against representative campaign data.
- [ ] Verify that weekly decisions remain traceable after references, ideas, or tasks are edited or archived.
- [ ] Verify keyboard navigation, readable mobile layouts, labeled controls, loading/empty/error states, and clear save confirmation.
- [ ] Verify staging deployment, database restore, image rollback, and the documented account/project deletion process.

Track adoption using completed user workflows: time to first project/reference, first published experiment with an observation, weekly-review completion, and a second campaign started from documented learning. Measure product value through better-supported decisions and available campaign outcomes; raw views alone do not establish success. Set numeric product targets after a small pilot provides a baseline.

## Later opportunities

- Official platform integrations for permitted metadata/metrics imports, with permission scopes and refresh behavior assessed per platform.
- Opt-in reminders for observation windows and weekly reviews.
- Reference recommendations based on a project's niche and user-saved examples, showing why each suggestion is relevant.
- Optional AI assistance for organizing references, summarizing comment themes, and drafting creator briefs, with source links and human review.
- Cross-campaign comparisons after measurement definitions are consistent.

These additions should follow a useful manual workflow. Automatic scraping, autonomous publishing, paid-ad management, billing, payment processing, and synthetic creator videos are not first-release commitments.

## Content provenance

The product workflow builds on the notebook's [audience](index.html#audience), [content formats](index.html#content), [creator operations](index.html#creators), [growth loop](index.html#iteration), and [30-day plan](index.html#plan) chapters.

The original materials are the supplied breakdowns of [the PlayKit/Julia Pintar discussion](https://www.youtube.com/watch?v=oyrQ-L4nziY) and [the organic 1,000-download experiment](https://www.youtube.com/watch?v=gUL6q-FndRE). Those videos were not independently transcribed for the notebook. The infrastructure, data model, feature milestones, and four-week adaptation in this document are product proposals, not claims made by the videos.
