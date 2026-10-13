# UX, Onboarding and the Admin Console for Caliban

*Research date: 2026-10-13. Scope: how the Caliban admin console (`web/`, React 19, Vite, Tailwind 4, TanStack Router and Query) and its documentation should get a non-technical user to a first answer, and how agents building the console should be guided. Grounded in the 2026-10-13 demo recordings (`recordings/2026-10-13/`, `shots/`).*

**Summary.** The 2026-10-13 demo worked, but a first answer needed about 12 concepts: models, tenants, PII modes and regex patterns, datasources and introspection, ontology review, routing intents and exemplars, node specs pasted as JSON, publish, golden sets, evals, promotion bars and gates, and API keys. Internal terms were on screen: CQIR, taint labels, surrogate scope, pass^k, kNN, trust tiers t0/t1. Two captions also had to admit "Admin API, no console page for this yet". The evidence points the same way across sources. First, ship a working example. Stripe starts every account in a sandbox, Metabase ships a sample database, Grafana ships TestData and Langfuse runs a demo project, so people see value before they configure anything. Second, give help when the user asks for it rather than through tours. NN/g found that tutorials "didn't improve task performance". Vendor data shows tours collapse past 4 or 5 steps (Chameleon: 72 to 74% completion at 3 or 4 steps, 16% at 7 or more), while tours the user starts complete at 67 to 69% against 23 to 31% for tours that start on their own. Third, restrict features early. The training-wheels study measured 21 to 52% faster task times with a reduced interface. Fourth, offer a form and a raw view of the same configuration, as AWS IAM, Cloudflare and Grafana do, instead of raw JSON alone. Caliban should therefore:
- **Split the console by persona.** One install checklist for the IT admin. One "build an assistant" path for the domain owner, starting from templates and an offline sample tenant.
- **Default or hide most of the 12 concepts.** Defaults cover routing, PII and gating; ontology review happens in bulk; evals come from "save this run as a test".
- **Rename the jargon.** Keep a single glossary with inline toggletips that link to the exact doc section.
- **Ship the docs with the binary.** Use Astro Starlight with Pagefind (both MIT), served offline at `/docs/<version>/`, with GitLab's self-hosted docs as the precedent.
- **Keep motion functional and mostly native.** CSS, Tailwind 4 and React 19.3's `<ViewTransition>` (with TanStack Router's `viewTransition`) cover most of it, with Carbon "productive" timings. Motion (MIT, about 4.6 kb with `LazyMotion` and `m`) is used only where native APIs fall short.
- **Steer coding agents with vendored guidance.** Copy Vercel's MIT web-interface-guidelines and parts of Anthropic's Apache-2.0 frontend-design skill into the repo. Do not install skills that fetch from the network or send telemetry.

---

## What the demo showed (grounding)

| Chapter screen | What a non-technical user faces | Evidence |
|---|---|---|
| Tenant (privacy) | "PII default mode", "Surrogate scope", "PII mode on sovereign models (trust tiers t0 and t1)", cache-hit price as a percentage of a flat price, config keys such as `[cache.semantic] enabled` in helper text | `shots/health-triage/08-evals.png` (the end-of-chapter shot lands on the tenant page) |
| Tenant (detectors, limits) | Rust regex syntax (`\bLB-\d{4}-\d{4}-\d\b`), six numeric caps (steps, tokens, wall clock, subnode depth, map fan-out, USD) shown at the same level as the basics | `shots/health-triage/03-tenant.png` |
| Ontology studio | 78 elements across seven kinds, each with its own version and approve or reject badge. The caption explains "typed CQIR ... lowered by a Rust compiler" | `shots/health-triage/05-ontology.png` |
| Routing intents | "caliban/auto classifies each request by its nearest exemplars"; the intent-to-node mapping can be seen but not edited ("Admin API, no console page for this yet") | `shots/health-triage/08-evals.png` |
| Node detail | Spec hash, vertex types (`verify`, `llm`), "Constrained decoding", "rag.\<collection\>:read scopes", "PII floor"; the spec is authored as JSON (`web/src/components/nodes/node-editor.tsx` starts from `NODE_SPEC_TEMPLATE`) | `shots/health-triage/07-node.png` |
| Evals | pass^3 against a "promotion bar"; the fintech gate passed only because its bar was 0.6 (README, "Real-model results") | `recordings/2026-10-13/README.md` |

The console is accurate and honest, and its helper text is already careful. The problem is order and exposure: everything an expert might tune is shown at first contact. The developer persona, by contrast, went from API key to first SDK call in about 20 seconds of video (chapter 21:51 to 22:10). The code path is already short. The console path is not.

---

## Sources

### Onboarding, time to first value, activation

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Stripe sandboxes ("Testing use cases")](https://docs.stripe.com/test-mode) | current | Every new account starts in a sandbox with test keys on the dashboard home and simulated objects; up to 5 isolated sandboxes, plus "Delete test data". | A sandbox tenant with sample data is the strongest pattern for a first answer before any real configuration. | **Adopt** |
| [Stripe, first API request with Stripe Shell](https://docs.stripe.com/get-started/api-request) | current | The first request runs inside the docs page, in the sandbox, then points to Workbench logs to see what happened. | "Try it" in the docs plus "see the trace it made" closes the loop. Caliban can do this against its own data plane, on-prem. | **Adapt** |
| [Stripe Workbench (vendor blog)](https://stripe.com/blog/workbench-a-new-way-to-debug-monitor-and-grow-your-stripe-integration) | 2024 | Inspector, Shell and API Explorer inside the dashboard, on by default; errors grouped with suggested fixes. | A model for a "Developer" drawer that shows the API call behind every console action. | **Adapt** |
| [Stripe docs with your keys, Markdoc (engineering blog)](https://stripe.dev/blog/markdoc) | 2022 | Docs personalized with the reader's test keys; Markdoc is open source (MIT) and validated in CI. | Personalize code samples with the instance URL and tenant **client-side**, with no egress. | **Adapt** |
| [Vercel Deploy Button](https://vercel.com/docs/deploy-button) | 2025 | One link clones a template and pre-configures project creation through query parameters. | Template links ("start from the triage template") can pre-fill Caliban's assistant wizard. | **Adapt** |
| [Supabase getting started](https://supabase.com/docs/guides/getting-started) and [database overview](https://supabase.com/docs/guides/database/overview) | current | Framework quickstarts; both a visual Table Editor and an SQL editor with saved snippets and quickstart scripts. | The same data reachable visually or as code, which is the dual-editor pattern. | **Adopt** (pattern) |
| [Linear start guide](https://linear.app/docs/start-guide) | current | A public demo workspace whose changes reset on refresh, plus separate admin guides. | A resettable demo tenant; separate admin material for the IT persona. | **Adopt** |
| [PostHog install wizard](https://posthog.com/docs/getting-started/install) | current | Setup is one command (`npx @posthog/wizard`) that edits the project. | Caliban SDK setup can be one command too, but the wizard calls a hosted AI. Any Caliban equivalent must use the tenant's own model. | **Adapt** |
| [Clerk keyless quickstart](https://clerk.com/docs/quickstarts/nextjs) and [blog](https://clerk.com/blog/accountless-setup-clerk-init-nextjs) (vendor blog) | 2024 to 2026 | `clerk init` works with no account. Missing keys raise an error that names the fix. | Errors that name the fix; first value before sign-up ceremony. | **Adopt** (error pattern) |
| [Retool templates](https://retool.com/templates) | current | One-click apps with sample data, usable as-is or as a starting point. | Assistant templates per use case (triage, legal research, support desk) that come with sample data. | **Adopt** |
| [Langfuse demo project](https://langfuse.com/docs/demo) | current | A live demo project in three steps: chat with a bot, open the trace it made, explore. | The exact loop Caliban needs: ask, see the run, see why. | **Adopt** |
| [Langfuse headless initialization](https://langfuse.com/self-hosting/administration/headless-initialization) | current | Environment variables create the org, project, keys and owner at boot. | The IT admin's install should create the first tenant, model and key from deploy config, with no console clicks needed. | **Adopt** |
| [LiteLLM Docker quick start](https://docs.litellm.ai/docs/proxy/docker_quick_start) | current | 6 steps: start, log in, add a model with **Test Connect**, test message, virtual key, call. | The closest on-prem peer. Its "Test Connect" button on model registration is cheap and valuable. | **Adopt** |
| [Portkey first request](https://portkey.ai/docs/introduction/make-your-first-request) and [model catalog](https://portkey.ai/docs/product/model-catalog) | current | Key, then SDK, then see the request in the dashboard; Integrations, Providers and Models as three layers. | Shows that even a SaaS gateway needs three layers to name a model. Caliban should hide the layers it can. | **Adapt** |
| [Helicone quick start](https://docs.helicone.ai/getting-started/quick-start) | current | A one-line base URL change; "first LLM request logged ... in under 2 minutes". Helicone is now [in maintenance mode under Mintlify](https://helicone.ai/blog/joining-mintlify) (2026). | Sets the bar for the developer path, which Caliban already meets. | **Adopt** (as the bar) |
| [OpenRouter quickstart](https://openrouter.ai/docs/quickstart) and [FAQ](https://openrouter.ai/docs/faq) | current | A free allowance plus a chat room in the browser before any key. | A "chat with this tenant" playground before any API key. | **Adopt** |
| [Metabase sample database](https://www.metabase.com/glossary/sample-database) | current | An H2 database of four tables ships with the product. | A precedent for **bundled, offline** sample data, which zero egress requires. | **Adopt** |
| [Grafana TestData](https://grafana.com/docs/grafana/latest/datasources/testdata/) | current | A built-in source of simulated data "with no external connection required". | A built-in sample datasource for Caliban (a small MongoDB or SQLite fixture). | **Adopt** |
| [Airtable templates](https://support.airtable.com/docs/using-airtable-templates) and [Notion templates](https://www.notion.com/help/start-with-a-template) | current | Templates arrive with sample data and a "clear data" action; Notion picks templates from onboarding answers. | Ask one question ("what do you want to build?") and pick the template from the answer. | **Adopt** |
| [Userpilot SaaS metrics benchmark](https://userpilot.com/saas-product-metrics/) (vendor report) | 2025 | Activation average 37.5%; time to value averages 1 day 12 h; onboarding checklist completion averages 19.2%, median 10.1%. | Checklists alone are not enough. They must be short and lead to value. | **Adapt** |
| [Appcues bite-sized checklists](https://goodux.appcues.com/behind-the-experience/appcues-bite-sized-checklists) (vendor case study) | n.d. | Splitting one long checklist into phased checklists raised completion from 2% to 25%. | Split setup into an "install" phase and a "first assistant" phase. | **Adopt** |
| [Lenny Rachitsky, good activation rate](https://www.lennysnewsletter.com/p/what-is-a-good-activation-rate) (newsletter survey) | 2022 | 500+ responses: average activation 34%, median 25%; SaaS average 36%. | A baseline for Caliban's own pilot metric (time to first answer). | **Adapt** |
| [Pendo feature adoption report](https://www.pendo.io/resources/the-2019-feature-adoption-report/) (vendor report) | 2019 | "80 percent of features ... rarely or never used"; 12% of features generate 80% of usage. | Supports hiding expert controls behind disclosure by default. | **Adapt** |
| [Amplitude B2B benchmarks](https://amplitude.com/blog/b2b-technology-product-benchmarks) (vendor report) | 2025 | 69% of products with strong early activation were strong at 3-month retention. | Ties early success in pilots to renewal. | **Adapt** |

### Guidance, disclosure, tours and empty states

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [NN/g, Progressive disclosure](https://www.nngroup.com/articles/progressive-disclosure/) | 2006 | Show the few important options first; "Designs that go beyond 2 disclosure levels typically have low usability". | Exactly two levels: basics, then "Advanced". Never nest drawers. | **Adopt** |
| [NN/g, Training wheels interface](https://www.nngroup.com/articles/training-wheels-user-interface/) (summarizing Carroll and Carrithers, 1984) | 2006 | A reduced interface gave 26% faster tasks and 69% more facts learned (study 1), and 52% faster advanced use (study 2). | The strongest quantitative case for a reduced domain-owner view. | **Adopt** |
| [NN/g, Wizards](https://www.nngroup.com/articles/wizards/) | 2017 | Use wizards for novices and infrequent setup; show all steps, specific button labels, save and resume. | One-off install and first-assistant creation, not everyday editing. | **Adopt** |
| [PatternFly wizard guidelines](https://www.patternfly.org/components/wizard/design-guidelines/) | current | Don't use a wizard for what a form handles; always end with a review step. | The assistant wizard ends on a review step that shows the generated spec. | **Adopt** |
| [NN/g, The power of defaults](https://www.nngroup.com/articles/the-power-of-defaults/) | 2005 | People follow defaults: 42% clicked the top hit, which still drew 34% when results were swapped. | Defaults are policy. Choose safe ones (reversible PII, tenant scope, gate on) and say so. | **Adopt** |
| [Carbon disclosures](https://carbondesignsystem.com/patterns/disclosures-pattern/) and [forms](https://carbondesignsystem.com/patterns/forms-pattern/) | current | Never hide what the task needs; one disclosure open at a time; helper text over placeholders and tooltips. | Rules for the "Advanced" section on tenant and node pages. | **Adopt** |
| [NN/g, Onboarding tutorials vs contextual help](https://www.nngroup.com/articles/onboarding-tutorials/) | 2023 | Tutorials are "push revelations" that interrupt; prefer "pull" help triggered by context. No numbers. | No launch tour. Contextual hints at the point of need. | **Adopt** |
| [NN/g, Mobile-app onboarding](https://www.nngroup.com/articles/mobile-app-onboarding/) | 2020 | "tutorials didn't improve task performance"; spend the effort on usability instead. | Fix the screens before adding guidance on top. | **Adopt** |
| [NN/g, Paradox of the active user](https://www.nngroup.com/articles/paradox-of-the-active-user/) | 1998 | Users start tasks immediately and skip manuals (Carroll and Rosson, 1987). | Help must live on the screen where the work happens. | **Adopt** |
| [NN/g, Empty states in complex applications](https://www.nngroup.com/articles/empty-state-interface-design/) | 2021 | Empty states should show system status, give learning cues and offer a direct path to the key task. | Every empty list in the console names one next action. | **Adopt** |
| [Carbon empty states](https://carbondesignsystem.com/patterns/empty-states-pattern/), [PatternFly empty state](https://www.patternfly.org/components/empty-state/design-guidelines/), [Atlassian empty state](https://atlassian.design/foundations/content/designing-messages/empty-state) | current | Types include "getting started" and "required configuration"; starter content; one CTA; one or two sentences, no jargon. | Ready-made copy rules for Caliban's `states.tsx`. | **Adopt** |
| [Chameleon benchmark 2024](https://www.chameleon.io/benchmark-report-24) (vendor report) | 2024 | Average tour completion 33.5%; five-step tours fall to 21.6%; tours launched from a checklist reach 64%. | If any tour exists, the user starts it and it is short. | **Adapt** |
| [Chameleon benchmark 2025](https://www.chameleon.io/benchmark-report) and [550M data points](https://www.chameleon.io/blog/mastering-product-tours) (vendor) | 2025 | Launcher tours 67% vs 31%; by length: 3 steps 72%, 4 steps 74%, 7+ steps 16% (blog figures, not in the report text). | Cap any guided walk at 4 steps. | **Adapt** |
| [Produktly onboarding benchmarks 2026](https://produktly.com/research/saas-onboarding-benchmarks-2026) (vendor report) | 2026 | Median tour completion 29%; 1 to 2 steps 73%, 3 to 5 steps 38%, 9+ steps 8%; user-initiated 69% vs auto-started 23% (correlation only). | Independent of Chameleon and in the same direction. | **Adapt** |
| Tour SaaS (Appcues, Pendo, Intercom, Chameleon) | n/a | Hosted widgets and analytics. | Ruled out by zero egress. Build the few hints needed in-house. | **Skip** |

### Forms, builders and language

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [AWS IAM visual and JSON editors](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html) | current | Switch between visual and JSON "anytime"; the visual editor may restructure the policy; the analyzer gives actionable fixes. | The model for node specs: a form first, a JSON tab for experts. | **Adopt** |
| [Cloudflare expression builder vs editor](https://developers.cloudflare.com/ruleset-engine/rules-language/expressions/edit-expressions/) | current | The builder shows a live text preview; when an expression is too advanced for the builder, it warns and lets the user stay in code. | The right way to handle specs the form cannot show (custom vertices, unusual guardrails). | **Adopt** |
| [Grafana dashboard JSON model](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/) | current | A JSON tab inside the same edit UI as the visual editor. | Same pattern, proven on-prem. | **Adopt** |
| [n8n AI workflow builder](https://docs.n8n.io/build/ways-of-building-workflows/ai-workflow-builder), [Retool Assist](https://docs.retool.com/apps/guides/assist/), [Zapier Copilot](https://help.zapier.com/hc/en-us/articles/23503999825421-Build-Zaps-faster-using-AI-powered-Copilot-Beta) (vendor) | 2024 to 2026 | Describe the goal; the tool drafts a config the user reviews and refines. None claims one-shot accuracy. | "Describe your assistant" runs on the tenant's own model and produces a **draft** spec that the user must review. | **Adapt** |
| [Dify DSL export](https://docs.dify.ai/en/use-dify/workspace/app-management) | current | Visual apps export as YAML DSL without secrets. | Specs as portable text, which Caliban already has; export must strip keys. | **Adopt** |
| [react-jsonschema-form](https://github.com/rjsf-team/react-jsonschema-form) (Apache-2.0) and [JSON Forms](https://github.com/eclipsesource/jsonforms) (MIT) | current | Forms generated from JSON Schema. | Caliban already syncs schemas (`pnpm sync:schemas`, ajv), but generated forms look generic. Use hand-built forms for the 5 to 10 common fields and the JSON tab for the rest. | **Skip** (as the main editor) |
| [NN/g error message guidelines](https://www.nngroup.com/articles/error-message-guidelines/) | 2023 | Put the message near its source, in human words, with a constructive fix; preserve input; don't say "invalid". | Rewrite control-plane validation errors (scope, cap and pin failures) into field-level messages. | **Adopt** |
| [GOV.UK error message](https://design-system.service.gov.uk/components/error-message/) and [validation](https://design-system.service.gov.uk/patterns/validation/) | 2024 | Say what went wrong and how to fix it; avoid "valid", "please", "sorry"; validate on submit and don't validate on blur. | Caliban's default should be validation on submit, with inline checks only where they help (regex, JSON). | **Adopt** |
| [Wroblewski, Inline validation (Etre study)](https://alistapart.com/article/inline-validation-in-web-forms/) | 2009 | 22 users: +22% success, -22% errors, -42% completion time with the best inline variant. Small sample. | Supports live testing of regex patterns against sample text, which conflicts with GOV.UK's on-submit rule; use inline checks only for syntax-heavy fields. | **Adapt** |
| [Google developer style guide, jargon](https://developers.google.com/style/jargon) | current | Replace jargon; if a term stays, define it in parentheses on first use or link to a definition. | The rule for CQIR, pass^k and the trust tiers. | **Adopt** |
| [GOV.UK clear language](https://guidance.publishing.service.gov.uk/writing-to-gov-uk-standards/writing-guidelines/clear-language/) | current | Plain English is mandatory; "80% of people preferred sentences written in clear English" in a legal-language study. | Experts also prefer plain language, so this helps the IT admin too. | **Adopt** |
| [Kimble, Michigan Bar Journal](https://www.michbar.org/journal/Details/ArticleID=5338) | 2026 | Judges and lawyers preferred plain versions by 80% to 86%. | Relevant to the law-firm buyer persona. | **Adopt** |
| [NN/g, Writing for domain experts](https://www.nngroup.com/articles/writing-domain-experts/) and [plain language for experts](https://www.nngroup.com/articles/plain-language-experts/) | 2017 | Experts accept shared vocabulary but scan and want brevity; 10th to 12th grade level for experts, sentences of 15 to 20 words. | Two text depths: a one-line plain summary, with the expert detail behind "Learn more". | **Adopt** |
| [Federal plain language guidelines (archived)](https://webarchive.library.unt.edu/web/20121006190813mp_/http:/www.plainlanguage.gov/howto/guidelines/FederalPLGuidelines/TOC.cfm) | 2011 | "Address separate audiences separately"; "Use the same term consistently". | One term per concept, enforced by a lint. | **Adopt** |
| [Microsoft style guide, simple words](https://learn.microsoft.com/en-us/style-guide/word-choice/use-simple-words-concise-sentences) | 2026 | One term per concept. | Same. | **Adopt** |
| [NN/g tooltip guidelines](https://www.nngroup.com/articles/tooltip-guidelines/), [Carbon toggletip](https://carbondesignsystem.com/components/toggletip/usage/), [Pickering, Tooltips and toggletips](https://inclusive-components.design/tooltips-toggletips/) (expert opinion) | 2017 to 2019 | Tooltips are a last resort and invisible to keyboard and touch users; toggletips are click-revealed buttons and may hold a link. | Glossary terms use toggletips holding a one-line definition and a "Learn more" link. | **Adopt** |
| [NN/g, UI copy: command names](https://www.nngroup.com/articles/ui-copy/) | 2019 | "Users shouldn't have to Google a command name". | "Make live", "Retire", "Propose elements" are fine; "Introspect" and "Promote" need plainer labels. | **Adopt** |

### AI platform patterns: traces, evals, approvals, cost

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Langfuse datasets](https://langfuse.com/docs/evaluation/dataset-runs/datasets) and [open source scope](https://langfuse.com/docs/open-source) (MIT core, `/ee` licensed) | current | "+ Add to dataset" on one observation, or bulk with field mapping; versioned datasets; experiments pin a version. | "Save this run as a test case" is how golden sets should be built: from real runs, not typed in. | **Adopt** |
| [Langfuse annotation queues](https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues) | current | "Complete + next" review flow with keyboard (1 to 9, Cmd+Enter, ? for help). | Use this for ontology review and the approval inbox: a queue with keys, not a wall of badges. | **Adopt** |
| [Langfuse LLM-as-a-judge](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge) | current | Evaluators from templates; variables mapped by clicking data; test on one sample before saving. | "Test on one sample" before running a whole eval avoids the judge failure the demo hit overnight (README, "Eval judge thinking"). | **Adopt** |
| [Braintrust experiments](https://www.braintrust.dev/docs/guides/experiments/interpret) and [playground](https://www.braintrust.dev/docs/guides/playground) (proprietary) | current | Diff against a baseline, "order by regressions", rows graded Improvement / Regression / Tradeoff / Tie. | The version-compare page should lead with regressions, not with a JSON-pointer diff. | **Adopt** (pattern) |
| [LangSmith compare experiments](https://docs.langchain.com/langsmith/compare-experiment-results), [annotation queues](https://docs.langchain.com/langsmith/annotation-queues), [datasets](https://docs.langchain.com/langsmith/manage-datasets-in-application) (proprietary) | current | Baseline plus red and green regression counts that filter the view; reviewers correct an output and "Add to Dataset". | Same; the corrected output becomes the golden expectation. | **Adopt** (pattern) |
| [Promptfoo web viewer](https://www.promptfoo.dev/docs/usage/web-ui/) (MIT, [now part of OpenAI](https://github.com/promptfoo/promptfoo)) | 2026 | A prompts by tests matrix; filters for Failures, Errors and Different; Cmd+K; state in the URL. | A failures-first filter and URL state for the eval results page. | **Adopt** (pattern) |
| [OpenAI evals guide](https://developers.openai.com/api/docs/guides/evals) and [deprecations](https://developers.openai.com/api/docs/deprecations) | 2026 | The hosted Evals platform becomes read-only 2026-10-31 and shuts down 2026-11-30. | Hosted eval UIs come and go; the eval UI belongs in the product, which Caliban already does. | **Skip** |
| [Humanloop](https://humanloop.com/) | 2025 | Team joined Anthropic; platform sunset (unavailable after 2025-09-08 per [W&B](https://wandb.ai/wandb_fc/product-announcements-fc/reports/Humanloop-is-sunsetting-Migrate-to-Weights-Biases-as-an-alternative--VmlldzoxMzk4ODc1Nw), a competitor blog). | No longer a reference. | **Skip** |
| [Arize Phoenix](https://github.com/Arize-ai/phoenix) | current | Self-hostable traces, datasets and experiments under Elastic License 2.0 (not OSI). | Pattern reference only; do not embed. | **Skip** (code) |
| [LangChain human-in-the-loop](https://docs.langchain.com/oss/python/langchain/human-in-the-loop) (MIT) and [Agent Inbox](https://github.com/langchain-ai/agent-inbox) (archived 2026-09-20) | 2026 | Decisions are approve, edit, reject (with feedback) and respond; one decision per action, in order. | The decision vocabulary for Caliban's inbox, with reject requiring a reason. | **Adopt** |
| [Inngest waitForEvent](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/wait-for-event) | current | Every human wait gets a timeout and an explicit expiry path. | Inbox items show "expires in" and what happens on expiry. | **Adopt** |
| [GitHub deployment reviews](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/reviewing-deployments) | current | Approve or reject with a comment; prevent self-review; admin bypass needs a comment. | Four-eyes approval for refunds and other writes. | **Adopt** |
| [Temporal Web UI](https://docs.temporal.io/web-ui) (MIT) | current | Saved, shareable filtered views; timeline, compact and JSON history views. | The run list gets saved views and a timeline-first run detail with a JSON tab. | **Adopt** |
| [OpenRouter activity](https://openrouter.ai/docs/guides/administration/activity-export), [Portkey analytics](https://portkey.ai/docs/product/observability/analytics), [LiteLLM cost tracking](https://docs.litellm.ai/docs/proxy/cost_tracking) | current | Cards for spend, tokens and requests; group by model, key or metadata; CSV export; cache-savings tab. | Caliban's Usage page should answer "who spent what on which assistant" in one grouping control and show cache savings. | **Adopt** |
| [Vercel spend management](https://vercel.com/docs/spend-management) and [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) | 2026 | Alerts at 50/75/100%, signed webhooks, an optional hard stop confirmed by typing the team name; both warn that alerts lag. | Caliban enforces caps before the call, so it can say "enforced, not estimated", which is a selling point. | **Adapt** |

### Keyboard, accessibility, headless components

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Superhuman, command palette (vendor blog)](https://blog.superhuman.com/how-to-build-a-remarkable-command-palette/) | 2021 | One shortcut everywhere; every feature is a command; show each command's shortcut; fuzzy search with aliases. | Aliases are how the old jargon still finds the renamed thing ("CQIR" finds "query plan"). | **Adopt** |
| [GitHub command palette](https://docs.github.com/en/get-started/accessibility/github-command-palette) | current | Cmd/Ctrl+K; prefixes for scopes; ? for help; shortcuts remappable for accessibility. | Prefixes for tenants, assistants and runs. | **Adopt** |
| [cmdk](https://github.com/pacocoursey/cmdk) (MIT) and [kbar](https://github.com/timc1/kbar) (MIT) | 2025 to 2026 | cmdk's last release was 1.1.1 (2025-03-14), and it pulls in Radix dialog; kbar 1.0.0 was released 2026-08. | cmdk is stable but slow-moving; a palette on a maintained headless combobox is safer. | **Skip** (cmdk), **Watch** (kbar) |
| [React Aria Components](https://react-aria.adobe.com/) (Apache-2.0), [Base UI](https://base-ui.com/) (MIT), [Radix](https://github.com/radix-ui/primitives) (MIT), [Ariakit](https://github.com/ariakit/ariakit) (MIT, paid "Plus" examples) | 2026 | Headless, APG-conformant primitives. React Aria is the most thorough on internationalization and screen readers. | The console hand-rolls dialog, tabs and drawer. Moving them to one headless library fixes focus handling once. | **Adopt** (React Aria or Base UI) |
| [WCAG 2.2](https://www.w3.org/TR/WCAG22/) and [what's new](https://www.w3.org/WAI/standards-guidelines/wcag/new-in-22/) | 2023/2024 | New AA criteria: 2.4.11 focus not obscured, 2.5.7 dragging, 2.5.8 target size 24 px, 3.3.8 accessible authentication; new A: 3.2.6 consistent help, 3.3.7 redundant entry. | Concrete checks listed under section 7. | **Adopt** |
| [EN 301 549 V4.1.1 (Deque, vendor blog)](https://www.deque.com/blog/en-301-549-v4-1-1-is-final-what-changed-what-it-means-and-what-you-should-do/) and [EAA](https://commission.europa.eu/strategy-and-policy/policies/justice-and-fundamental-rights/disability/union-equality-strategy-rights-persons-disabilities-2021-2030/european-accessibility-act_en) | 2026 | EN 301 549 V4.1.1 (published 2026-09-02) adopts WCAG 2.2 AA; the EAA has applied since 2025-06-28. | B2B on-prem is probably outside the EAA's consumer scope (our inference), but public and regulated buyers ask for EN 301 549 conformance. | **Adopt** |
| WAI-ARIA APG: [grid](https://www.w3.org/WAI/ARIA/apg/patterns/grid/), [treegrid](https://www.w3.org/WAI/ARIA/apg/patterns/treegrid/), [combobox](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/), [modal dialog](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/), [tabs](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/) | current | Keyboard and ARIA contracts for each widget. | Treegrid for the run span tree, combobox for the palette, alert dialog for Retire and Delete. | **Adopt** |

### Motion

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [NN/g, animation purpose](https://www.nngroup.com/articles/animation-purpose-ux/) and [duration](https://www.nngroup.com/articles/animation-duration/) | 2020 | Motion is for feedback, state change, spatial continuity and signifiers; 100 to 400 ms; enter slightly slower than exit; most animations run too long. | The full brief for an admin console. | **Adopt** |
| [IBM Carbon motion](https://carbondesignsystem.com/elements/motion/overview/) | current | "Productive" motion for tools: 70, 110, 150, 240, 400, 700 ms; productive curves such as `cubic-bezier(0.2, 0, 0.38, 0.9)`. | Caliban is a work tool, so it takes the productive set. | **Adopt** |
| [Material 3 motion tokens (token source)](https://github.com/material-components/material-web/blob/main/tokens/versions/v0_192/_md-sys-motion.scss) and [M3 Expressive springs](https://m3.material.io/blog/m3-expressive-motion-theming) | 2025 | Durations 50 to 1000 ms; "standard" easing `cubic-bezier(0.2, 0, 0, 1)`; Expressive moves to springs. | A cross-check on Carbon. Springs add expressiveness an admin console doesn't need. | **Adapt** |
| [Apple HIG, motion](https://developer.apple.com/design/human-interface-guidelines/motion) | current | Motion must be purposeful and optional, never the only carrier of meaning; avoid motion on frequent interactions. | No motion on table row hover or other high-frequency actions. | **Adopt** |
| [Fluent 2 motion](https://fluent2.microsoft.design/motion) | current | Functional, natural, consistent; larger elements get more time. | Same direction; no numbers on the page. | **Adapt** |
| [Heer and Robertson, animated transitions](https://idl.uw.edu/papers/animated-transitions) | 2007 | Peer-reviewed: animated transitions between chart states can significantly improve graphical perception when they are simple and staged. | Animate chart changes on the Usage page when the range or grouping changes. | **Adopt** |
| [Chevalier et al., staggered transitions](https://visualcognition.psych.northwestern.edu/publications/ChevalierDragicevicFranconeri2014.pdf) | 2014 | Peer-reviewed: staggering gave negligible or negative benefit for tracking objects. | No staggered list entrances. | **Adopt** |
| [MDN prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion), [WCAG 2.3.3](https://www.w3.org/WAI/WCAG22/Understanding/animation-from-interactions.html), [WCAG 2.2.2](https://www.w3.org/WAI/WCAG22/Understanding/pause-stop-hide.html) | current | Replace large motion with opacity under reduce; auto-updating content needs pause (2.2.2 is Level A). | Live run and usage views need a pause control. | **Adopt** |
| [Motion](https://motion.dev/docs/react-reduce-bundle-size) ([repo](https://github.com/motiondivision/motion), MIT; paid [Motion+](https://motion.dev/plus) extras) | 2026 | Full `motion` component about 34 kb; `LazyMotion` + `m` about 4.6 kb initial, plus about 15 kb for `domAnimation`; `useAnimate` mini about 2.3 kb (vendor figures). | The JS fallback for the few things native CSS cannot do well, such as layout animation of the drawer and inbox removal. | **Adopt** (narrowly) |
| [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transition_API) ([Baseline 2025-10-14](https://github.com/web-platform-dx/web-features/blob/main/features/view-transitions.yml)), [React `<ViewTransition>`](https://react.dev/reference/react/ViewTransition) ([stable in 19.3.0](https://github.com/facebook/react/releases/tag/v19.3.0)), [TanStack Router `viewTransition`](https://tanstack.com/router/latest/docs/framework/react/api/router/NavigateOptionsType) | 2025 to 2026 | Same-document transitions ship in Chrome 111, Safari 18 and Firefox 144; React 19.3 makes `<ViewTransition>` stable; the router option wraps navigations and is ignored where unsupported. | Zero bytes of dependency for route and list-to-detail continuity; the console already depends on React ^19.3.0. | **Adopt** |
| [Tailwind 4 animation](https://tailwindcss.com/docs/animation) (MIT) and [tw-animate-css](https://github.com/Wombosvideo/tw-animate-css) (MIT) | 2026 | `motion-safe:` and `motion-reduce:` variants and `@theme` keyframes; tw-animate-css adds enter and exit utilities in pure CSS. | Motion tokens live in `@theme` next to color tokens. | **Adopt** (Tailwind), **Adapt** (tw-animate-css, v2 breaking changes announced) |
| [AutoAnimate](https://auto-animate.formkit.com/) (MIT, about 3.2 kB gzip per Bundlephobia) | 2026 | Zero-config list add, remove and move; respects reduced motion. | A cheap option for the inbox and review queue lists. | **Adapt** |
| [react-spring](https://github.com/pmndrs/react-spring) (MIT, about 20 kB gzip) | 2026 | Spring physics. | More than a console needs. | **Skip** |
| [GSAP Standard "No Charge" license](https://gsap.com/standard-license/) | 2025 | Free but proprietary (Webflow), revocable, not OSI. | Fails an on-prem OSI license policy. | **Skip** |

### Documentation

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Diataxis](https://diataxis.fr/) | current | Four kinds of docs: tutorials, how-to guides, reference, explanation. | The structure for user docs; the research notes are "explanation", the OpenAPI spec is "reference". | **Adopt** |
| [NN/g, Help and documentation](https://www.nngroup.com/articles/help-and-documentation/) | 2020 | Proactive vs reactive help; contextual help should be brief and tied to the task. | "Learn more" links from each field to its exact section. | **Adopt** |
| [PostHog, how we write docs](https://posthog.com/handbook/content/docs) (vendor handbook) | current | "We treat our docs like a product". | Docs get owners, reviews and a definition of done per feature. | **Adopt** |
| [Astro Starlight](https://starlight.astro.build/guides/site-search/) (MIT) and [Starlight i18n](https://starlight.astro.build/guides/i18n/) | 2026 | Pagefind search by default, built into the static output; i18n with fallback and RTL. | The website is Astro already. Starlight gives offline search and Arabic and French support (law and health demos) at no cost. | **Adopt** |
| [Pagefind](https://pagefind.app/) (MIT) | 2026 | Static, chunked search index; a 10,000-page site stays "under 300kB". | Search that works in an air gap. | **Adopt** |
| [GitLab self-hosted docs](https://docs.gitlab.com/administration/docs_self_host/) | current | Docs ship as a versioned image or static files; the in-app `/help` URL points at it and includes the version. | The precedent for Caliban: docs versioned with the binary, links pinned to the running version. | **Adopt** |
| [Elastic air-gapped install](https://www.elastic.co/guide/en/elastic-stack/8.9/air-gapped-install.html) | 8.x | Offline docs only on request for supported customers. | The weaker model; avoid it. | **Skip** |
| [Grafana configuration](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/) | current | Switches for update checks, news feed, gravatar, feedback links and help. | A checklist of egress points the console must never have. | **Adopt** (as checklist) |
| [Docusaurus versioning](https://docusaurus.io/docs/versioning) (MIT) | 2026 | Snapshot docs per minor version, keep fewer than about 10, none per patch release. | Versioning policy, applied in Starlight. | **Adapt** |
| [Material for MkDocs in maintenance mode](https://squidfunk.github.io/mkdocs-material/blog/2025/11/05/zensical/) | 2025 | Critical fixes only, for at least 12 months; successor is Zensical. | Do not start on it. | **Skip** |

### UI and UX guidance for coding agents

| Source | Year | Core idea | Why it matters for Caliban | Verdict |
|---|---|---|---|---|
| [Anthropic frontend-design skill](https://github.com/anthropics/skills/tree/main/skills/frontend-design) (Apache-2.0) and [blog (vendor)](https://claude.com/blog/improving-frontend-design-through-skills) | 2025 to 2026 | Avoid "distributional convergence"; plan tokens first; visible focus and reduced motion as a floor; motion mainly in answer to user actions; consistent verbs ("Publish" leads to "Published"); errors that don't apologize; flags spaced em dash labels as a generated-design tell. | Leans toward marketing pages, but the writing, motion and quality-floor rules fit the console. | **Adapt** (vendor the relevant rules) |
| [Vercel web-interface-guidelines](https://github.com/vercel-labs/web-interface-guidelines) (MIT) | 2025 to 2026 | Checkable rules: keyboard everywhere, `:focus-visible`, 24 px targets, URL as state, confirm or undo destructive actions, `aria-live` toasts, animate only transform and opacity, never `transition: all`, interruptible animations, design all states, `tabular-nums`, errors next to fields, don't pre-disable submit. | The best fit for an admin console; usable as review criteria. | **Adopt** (vendored copy) |
| [Vercel agent-skills: web-design-guidelines](https://github.com/vercel-labs/agent-skills/tree/main/skills/web-design-guidelines) (MIT in skill files) | 2026 | Review skill that fetches its rules from GitHub on each run. | Network fetch at run time; use the vendored guidelines instead. | **Skip** |
| [Vercel agent-skills: react-best-practices](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-best-practices) (MIT) | 2026 | 70 rules; many are Next.js and RSC specific. | The bundle, re-render and rendering rules apply to a Vite SPA. | **Adapt** |
| [Vercel agent-skills: react-view-transitions](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions) (MIT) | 2026 | Every view transition must express a spatial relationship; no slides between tabs. | Good rules, but stale: says to install canary, whereas React 19.3 is stable. | **Adapt** |
| [Impeccable](https://github.com/pbakaus/impeccable) (Apache-2.0) | 2025 to 2026 | One skill with audit, critique, polish and animate commands, deterministic detectors, PRODUCT.md and DESIGN.md; timing table (100 to 150 ms feedback, 150 to 300 ms state, 300 to 500 ms layout); "operate" surfaces keep motion for feedback and continuity. The CLI calls a hosted API and sends opt-out telemetry. | The website's `PRODUCT.md` already uses its schema. Adopt the reference text; do not run the CLI in the zero-egress build path. | **Adapt** |
| [Google DESIGN.md](https://github.com/google-labs-code/design.md) (Apache-2.0) | 2026 | Token spec in YAML front matter plus rationale; a linter checks references and WCAG contrast. | `website/DESIGN.md` already follows this shape; give the console one too. | **Adopt** |
| [shadcn skill](https://github.com/shadcn-ui/ui/tree/main/skills/shadcn) (MIT) | 2026 | Semantic tokens only, `gap` over `space-*`, accessible field markup, a Title on every dialog, Skeleton and Empty components. Runs `npx shadcn` and searches registries at load. | Some rules transfer; Caliban does not use shadcn, and the skill needs the network. | **Skip** (install), **Adapt** (two or three rules) |
| [UI UX Pro Max](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) (MIT) | 2025 to 2026 | CSV catalogs of styles, palettes and Google Fonts pairings. | Trend catalog, not principles; font imports mean egress. | **Skip** |
| [web-artifacts-builder](https://github.com/anthropics/skills/tree/main/skills/web-artifacts-builder) (Apache-2.0) | 2025 | React 18, Tailwind 3 and Parcel, bundled into single-file artifacts. | Wrong stack and purpose. | **Skip** |
| [ComposioHQ](https://github.com/ComposioHQ/awesome-claude-skills) and [travisvn](https://github.com/travisvn/awesome-claude-skills) awesome lists | 2026 | Indexes, no license detected. | Discovery only. | **Skip** |

---

## 1. Time to first value: two personas, two short paths

The demo mixed two jobs. One person installs Caliban once: models, SSO, provider keys, deployment defaults. Another person, repeatedly, builds an assistant over their own data. Federal plain language guidance says to "address separate audiences separately" ([archived guidelines](https://webarchive.library.unt.edu/web/20121006190813mp_/http:/www.plainlanguage.gov/howto/guidelines/FederalPLGuidelines/TOC.cfm)). The training-wheels results ([NN/g](https://www.nngroup.com/articles/training-wheels-user-interface/)) show that a reduced interface speeds up novices and does not slow their later expert use.

**IT admin (installs once).**
- **Headless first boot.** Follow Langfuse's model ([headless init](https://langfuse.com/self-hosting/administration/headless-initialization)): the deploy config (compose env, Helm values, air-gap bundle) can declare the first model endpoint, the first tenant with safe defaults, and an admin user. After `up`, the console opens on a running system, not a blank one.
- **An "Install" checklist on Overview with at most 5 items.** For example: model reachable, sign-in configured, a tenant exists, privacy defaults reviewed, docs reachable. Each item has a "Check" button, like LiteLLM's **Test Connect** ([quick start](https://docs.litellm.ai/docs/proxy/docker_quick_start)). The checklist is derived from live state (green when true), so it can never drift from reality. Short phased checklists beat long ones: Appcues went from 2% to 25% completion by splitting one ([case study, vendor](https://goodux.appcues.com/behind-the-experience/appcues-bite-sized-checklists)), and average checklist completion is only 19.2% ([Userpilot 2025, vendor](https://userpilot.com/saas-product-metrics/)).

**Domain owner (builds assistants).**
- **One page, "Build an assistant", in 4 steps:**
  1. What should it do (pick a template or describe it).
  2. Which data (pick an existing datasource or upload documents).
  3. Try it (a chat panel against the draft; every answer links to its run).
  4. Share it (create a key, or attach it to `caliban/auto`).
- **Everything else defaults.** That includes the intent (named after the assistant and seeded with the template's exemplars), the PII mode (inherited from the tenant), the budget caps (tenant defaults) and the eval gate (on, with the template's bar). Four steps is also where tour completion stays above 70% in the vendor data ([Chameleon, vendor](https://www.chameleon.io/blog/mastering-product-tours)).

**Sample tenant, bundled offline.**
- **A `sandbox` tenant created on first boot from fixtures inside the image.** It holds a small MongoDB or SQLite catalogue, a dozen documents and one published assistant with a golden set. This is the Metabase and Grafana TestData pattern ([Metabase](https://www.metabase.com/glossary/sample-database), [Grafana](https://grafana.com/docs/grafana/latest/datasources/testdata/)) and the Langfuse demo loop ([demo](https://langfuse.com/docs/demo)): ask, open the run, see why.
- **A reset action, and one-click removal for production.** Reset works like Linear's demo, where changes reset on refresh ([start guide](https://linear.app/docs/start-guide)), and like Stripe's "Delete test data" ([sandboxes](https://docs.stripe.com/test-mode)). The sandbox can be deleted in one click for production installs.
- **The four demo scenarios (health triage, law, developer, fintech) become the template set.** Their specs, golden cases and intents already exist in the deploy tree.

**What a domain owner no longer sees by default:**

| Demo concept | New default | Where the expert finds it |
|---|---|---|
| Routing intents and exemplars | Created from the assistant; exemplars from the template, plus a "this went to the wrong assistant" button on runs (the developer demo's runtime fix) | Assistant > Routing (Advanced) |
| Node spec JSON | Generated by the wizard | Assistant > Spec tab (JSON) |
| Publish vs make live vs promote | One "Go live" button that publishes, runs the evals and promotes if the bar passes; the steps are shown as progress, not as three buttons | Versions table (Advanced) |
| Golden sets | Template cases plus "Save this run as a test" | Assistant > Tests |
| Promotion bar | Template default (0.8); changing it below the default asks for a reason, which goes to the audit log | Assistant > Tests > Settings |
| Ontology review of 78 to 89 elements | Bulk accept of high-confidence proposals; review only what the assistant uses | Ontology studio |
| PII regex patterns | Built-in detectors plus a "pattern from examples" helper with live test (see section 4) | Tenant > Privacy > Advanced |
| Node caps (steps, tokens, depth, fan-out) | Tenant defaults, untouched | Tenant > Limits (Advanced) |

## 2. Progressive disclosure and defaults

- **Two levels only** ([NN/g](https://www.nngroup.com/articles/progressive-disclosure/)): the basics, then one "Advanced" section per page. Never put a disclosure inside a disclosure ([Carbon](https://carbondesignsystem.com/patterns/disclosures-pattern/)). Use the drawer component the console already has (`components/ui/drawer.tsx`) only for side tasks, not for settings.
- **Classify every field before moving it.** Use frequency and consequence, the task-analysis split NN/g recommends. On the tenant page, for example:
  - **Basic:** PII mode, the data each assistant may use, and spend caps.
  - **Advanced:** surrogate scope, PII on sovereign models, cache-hit price, regex patterns, the six node caps.
  - **Exception:** "Advanced" must not hide anything needed to complete the task (Carbon). A tenant that will quote phone numbers needs reversible PII, so the mode stays basic and its consequence is spelled out.
- **Role-based navigation.** The sidebar today shows 15 sections to every role (screenshots). A domain-owner role should see Assistants, Data, Runs, Inbox, Usage and Help. The admin role adds Models, Tenants, Users and roles, Audit, Settings and Tools. `lib/permissions.ts` already exists to hang this on. Pendo's data that most features are rarely used ([vendor](https://www.pendo.io/resources/the-2019-feature-adoption-report/)) supports hiding by role rather than by clutter.
- **Defaults are policy and should say so** ([NN/g, defaults](https://www.nngroup.com/articles/the-power-of-defaults/)). Each defaulted control shows "Default: reversible, recommended because ..." and a "Reset to default" link (the node limits card already has a "Defaults" badge; generalize it).
- **Close the two console gaps the demo exposed.** Editing intent-to-node routes and the tenant default route were API only ("Admin API, no console page for this yet" in the captions of `08-evals.png` and `03-tenant.png`). A UX plan that relies on defaults still needs a way to change them.

## 3. Guided help: what to build and what not to

Ranked by evidence:

1. **Empty states that teach** ([NN/g](https://www.nngroup.com/articles/empty-state-interface-design/), [PatternFly types](https://www.patternfly.org/components/empty-state/design-guidelines/), [Atlassian copy rules](https://atlassian.design/foundations/content/designing-messages/empty-state)).
   - **Write each one from the same parts:** what this is (one sentence), one primary action and one "Learn more" link.
   - **Use the "required configuration" type** when a prerequisite is missing. For example: "This assistant has no data yet. Connect a datasource".
   - **The ontology review queue already does this well.** "Nothing to review. Run Introspect on a datasource to propose new entities, metrics and policies" (`05-ontology.png`). Only "Introspect" needs a plainer word ("Scan").
2. **Phased checklists, derived from state** (section 1).
3. **Contextual pull hints.** Use toggletips on glossary terms and inline helper text, which the console already writes well ([NN/g](https://www.nngroup.com/articles/onboarding-tutorials/), [Carbon toggletip](https://carbondesignsystem.com/components/toggletip/usage/)). For first-time hints, show one dismissible hint per page and store it per user on the server, because `localStorage` is per browser.
4. **No automatic product tour.** NN/g finds tutorials do not improve task performance ([mobile onboarding](https://www.nngroup.com/articles/mobile-app-onboarding/)). The vendor data is consistent:
   - auto-started tours complete at 23 to 31%, against 67 to 69% for tours the user starts ([Chameleon 2025](https://www.chameleon.io/benchmark-report), [Produktly 2026](https://produktly.com/research/saas-onboarding-benchmarks-2026), both vendor);
   - five-step tours fall to 21.6% ([Chameleon 2024](https://www.chameleon.io/benchmark-report-24)).

   If a walkthrough is ever built, it is user-started from the checklist, at most 4 steps, and in-house (tour SaaS is ruled out by zero egress).
5. **Templates and the sandbox are guidance too** (NN/g lists templates and wizards as proactive help in [Help and documentation](https://www.nngroup.com/articles/help-and-documentation/)). They teach by example and do not interrupt.

## 4. Forms and builders instead of raw JSON

- **Assistant spec: form first, JSON tab second.** Follow the AWS IAM, Cloudflare and Grafana model ([IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html), [Cloudflare](https://developers.cloudflare.com/ruleset-engine/rules-language/expressions/edit-expressions/), [Grafana](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/)).
  - **The form covers the fields that matter to a domain owner:** name, purpose, the model's instructions, data it may read, tools it may call, which actions need approval, answer style and test cases.
  - **The JSON tab is the existing `node-editor.tsx`**, kept as is with its schema checks.
  - **When a spec uses something the form cannot show,** the form displays "This version uses settings only the JSON view can edit" and stays read-only for those fields (Cloudflare's behavior). It never silently drops them.
- **"Describe your assistant."** The wizard's first step can draft a spec from a paragraph, using the tenant's own sovereign model through Caliban itself (no egress). Every comparable product presents the result as a draft to review, not a final config ([n8n](https://docs.n8n.io/build/ways-of-building-workflows/ai-workflow-builder), [Retool Assist](https://docs.retool.com/apps/guides/assist/), [Zapier Copilot](https://help.zapier.com/hc/en-us/articles/23503999825421-Build-Zaps-faster-using-AI-powered-Copilot-Beta)). Caliban should show the draft in the form, highlight the generated fields and require the review step PatternFly prescribes ([wizard](https://www.patternfly.org/components/wizard/design-guidelines/)). Because the draft goes through the normal publish checks and eval gate, a bad generation cannot go live unnoticed.
- **PII patterns without regex.**
  1. "Add a pattern from examples": paste two or three real identifiers and the console proposes a pattern.
  2. A live test box shows matches in sample text, using the same Rust-regex rules (no lookaround), validated server-side.
  3. Raw regex stays under Advanced.

  Live feedback is justified here by the inline validation study, +22% success and -42% time ([Wroblewski, small study](https://alistapart.com/article/inline-validation-in-web-forms/)). Elsewhere, GOV.UK's rule of validating on submit is the default ([validation](https://design-system.service.gov.uk/patterns/validation/)).
- **Error messages.** Control-plane rejections (scope not granted, cap exceeded, pin missing) should map to the field that caused them. Each message says what happened and how to fix it, keeps the user's input, and avoids "invalid" ([NN/g](https://www.nngroup.com/articles/error-message-guidelines/), [GOV.UK](https://design-system.service.gov.uk/components/error-message/)). `lib/errors.ts` is the place for a code-to-message table. As with Clerk, an error should name the fix ([Clerk keyless](https://clerk.com/blog/accountless-setup-clerk-init-nextjs)).
- **Not JSON Schema form generators.** [rjsf](https://github.com/rjsf-team/react-jsonschema-form) (Apache-2.0) and [JSON Forms](https://github.com/eclipsesource/jsonforms) (MIT) would render every schema field and recreate the exposure problem. Hand-build the 5 to 10 fields people need.

## 5. Plain language and microcopy

- **One glossary file is the source of truth.** Every internal term gets a UI label, a one-line definition, a doc anchor and aliases (the old terms) for search and the palette. `lib/labels.ts` already maps enums to labels and hints (`TRUST_TIER_LABEL`, `TRUST_TIER_HINT`); extend it rather than invent a new system. Use one term per concept ([Microsoft](https://learn.microsoft.com/en-us/style-guide/word-choice/use-simple-words-concise-sentences), [federal guidelines](https://webarchive.library.unt.edu/web/20121006190813mp_/http:/www.plainlanguage.gov/howto/guidelines/FederalPLGuidelines/TOC.cfm)).
- **Proposed renames** (to validate with users, see open questions):

| Internal term | UI label | One-line definition |
|---|---|---|
| Node | Assistant (for nodes with a model step); "Step" for vertices | An assistant answers one kind of request using your data and tools. |
| CQIR | Query plan | The structured query the model picks; Caliban turns it into a safe database query. |
| Ontology / semantic model | Data map | The business names for your data that answers are allowed to use. |
| Introspect | Scan | Read the structure of a datasource and suggest entries for the data map. |
| Intent, exemplars, kNN | Request type, example requests | Caliban sends a request to the assistant whose examples it most resembles. |
| pass^k | "Passed all 3 tries" (k shown as the number) | A test counts only if it passes every time it is run. |
| Promotion bar / gate | Release bar | The share of tests that must pass before a version can go live. |
| Taint label / untrusted | Untrusted (from outside) | Text that came from a document, user or tool and must not be obeyed as instructions. |
| Surrogate scope | Stand-in consistency: per tenant / per request | Whether the same personal data gets the same stand-in across requests. |
| Trust tiers t0 to t3 | On your servers / Attested cloud / Contracted provider / Public API | Where the model runs and what guarantees you have. |
| PII mask / reversible | Hide permanently / Hide and restore | Whether personal data is restored in the answer the caller sees. |

- **Toggletips for terms that stay.** Use a button showing the definition and a "Learn more" link to the doc anchor ([Carbon](https://carbondesignsystem.com/components/toggletip/usage/), [Pickering](https://inclusive-components.design/tooltips-toggletips/)). Do not use hover-only tooltips ([NN/g](https://www.nngroup.com/articles/tooltip-guidelines/)). Define a kept term in parentheses on first use ([Google style guide](https://developers.google.com/style/jargon)).
- **Keep config keys out of default helper text.** Strings like `[routing] auto_cache_hit_fraction` belong behind "Learn more" or in the Advanced section; the domain owner never edits the config file.
- **Lint the copy.** A unit test fails when a banned term (CQIR, kNN, taint, pass^k, vertex, surrogate) appears in a non-Advanced UI string. This also enforces the no-em-dash rule in UI copy.
- **Two depths of text.** A plain one-line summary, with the expert detail behind "Learn more" ([NN/g, domain experts](https://www.nngroup.com/articles/writing-domain-experts/)). Experts also prefer plain versions, by 80% ([GOV.UK](https://guidance.publishing.service.gov.uk/writing-to-gov-uk-standards/writing-guidelines/clear-language/)) to 86% ([Kimble](https://www.michbar.org/journal/Details/ArticleID=5338)), so this does not cost the IT admin anything.

## 6. AI-platform interaction patterns

- **Runs.** Lead the run detail with the plain story:
  > "Classified as *triage* (example match 0.91) → asked 2 questions of the data → answered in 12.5 s, cost $0.004, personal data hidden: 3 items."

  Below it, a timeline of steps as an APG treegrid ([treegrid](https://www.w3.org/WAI/ARIA/apg/patterns/treegrid/)), and a JSON tab, Temporal-style ([Web UI](https://docs.temporal.io/web-ui)). The run list gets saved filter views stored in the URL ([Promptfoo](https://www.promptfoo.dev/docs/usage/web-ui/), Temporal).
- **"Save as test" on every run.** Follow Langfuse's "+ Add to dataset" and LangSmith's "Add to Dataset" ([Langfuse](https://langfuse.com/docs/evaluation/dataset-runs/datasets), [LangSmith](https://docs.langchain.com/langsmith/manage-datasets-in-application)). The domain owner corrects the answer if needed, and the corrected answer becomes the expectation. Golden sets then grow from real traffic, and a misroute becomes a routing test with one click. The developer demo's runtime exemplar fix is the same idea.
- **Eval results.**
  - **Failures first:** filters for Failures, Errors and Changed ([Promptfoo](https://www.promptfoo.dev/docs/usage/web-ui/)).
  - **Version compare** against the live version, "order by regressions", with red and green counts that filter ([Braintrust](https://www.braintrust.dev/docs/guides/experiments/interpret), [LangSmith](https://docs.langchain.com/langsmith/compare-experiment-results)). The current JSON-pointer diff (`07-node.png`, "Compare versions") becomes the Advanced view.
  - **"Test the judge on one case"** before a full run ([Langfuse](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)). The overnight judge failure would have shown up in seconds.
- **Approval inbox.**
  - **Decisions:** approve, edit arguments, or reject with a required reason ([LangChain HITL](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)).
  - **Context:** each item shows "why this needs you" in one line (for example "Refund write, the request contained untrusted text"), the arguments compared with the account on file (the fintech R2 case), and an expiry with its consequence ([Inngest](https://www.inngest.com/docs/features/inngest-functions/steps-workflows/wait-for-event)).
  - **Controls:** an optional rule that blocks the requester from approving their own item ([GitHub](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/reviewing-deployments)), and keyboard review ([Langfuse queues](https://langfuse.com/docs/evaluation/evaluation-methods/annotation-queues)).
- **Ontology review as a queue.** "Accept all suggestions above 0.9 confidence" on the entities and fields used by the selected assistant, then a keyboard queue for the rest (A accept, R reject, E edit, arrows to move). The current page already sorts "least confident first"; the batch step is what is missing.
- **Usage and cost.** Put cards for spend, requests and cache savings first, then one "group by" control (assistant, key, model, request type) and CSV export ([OpenRouter](https://openrouter.ai/docs/guides/administration/activity-export), [Portkey](https://portkey.ai/docs/product/observability/analytics), [LiteLLM](https://docs.litellm.ai/docs/proxy/cost_tracking)). Show budget alerts at 50, 75 and 100% in the console ([Vercel](https://vercel.com/docs/spend-management)). Caliban checks caps before every model call, so the UI can truthfully say "enforced", unlike AWS's delayed alerts ([AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)).

## 7. Command palette, keyboard, accessibility

- **Palette.** Cmd/Ctrl+K everywhere, built on a maintained headless combobox (React Aria Autocomplete or Base UI Combobox; [React Aria](https://react-aria.adobe.com/), Apache-2.0; [Base UI](https://base-ui.com/), MIT) following the APG combobox contract ([APG](https://www.w3.org/WAI/ARIA/apg/patterns/combobox/)). Contents: navigation, "create assistant", "open run by id", tenant switching, and glossary aliases so old terms still work ([Superhuman, vendor](https://blog.superhuman.com/how-to-build-a-remarkable-command-palette/)). Show each command's shortcut and add a `?` cheat sheet ([GitHub](https://docs.github.com/en/get-started/accessibility/github-command-palette)). cmdk is MIT but slow-moving (last release 2025-03-14) and depends on Radix ([cmdk](https://github.com/pacocoursey/cmdk)).
- **One headless library for dialogs, tabs, drawer and menus.** The console hand-rolls these (`components/ui/`). Moving them onto one library gets focus traps, focus return and keyboard behavior right everywhere ([APG dialog](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/), [tabs](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/)). Destructive actions (Retire, Delete key) use an alert dialog with focus on the safe option.
- **WCAG 2.2 AA checks that matter here** ([WCAG 2.2](https://www.w3.org/TR/WCAG22/)):
  - 2.5.8 **target size** of at least 24 by 24 px. Watch the copy-hash, trash and inline badge buttons ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html)).
  - 2.4.11 **focus not obscured**. The demo caption bar and any sticky header or footer need `scroll-padding` ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/focus-not-obscured-minimum.html)).
  - 3.3.8 **accessible authentication**. The admin token field must allow paste and password managers ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/accessible-authentication-minimum.html)).
  - 3.3.7 **redundant entry**. The wizard reuses tenant and datasource choices instead of asking again ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/redundant-entry.html)).
  - 3.2.6 **consistent help**. The Help entry sits in the same place on every page ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/consistent-help.html)).
  - 2.5.7 **dragging**. Any future graph editor needs click alternatives ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/dragging-movements.html)).
  - 4.1.3 **status messages**. "Detectors saved." and "Limits saved." use `role="status"`; errors use `role="alert"`, sparingly ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/status-messages.html)).
  - 1.4.11 **non-text contrast** of 3:1 for badge borders and chart lines ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html)). The pale approved and rejected badges in `05-ontology.png` should be checked, and status must never be shown by color alone.
  - 1.4.10 **reflow**. Tables are exempt but their filters and pagination are not ([understanding](https://www.w3.org/WAI/WCAG22/Understanding/reflow.html)).
  - 2.2.2 **pause** for live-updating runs and usage (Level A, [understanding](https://www.w3.org/WAI/WCAG22/Understanding/pause-stop-hide.html)).
- **Procurement.** EN 301 549 V4.1.1 adopts WCAG 2.2 AA ([Deque, vendor blog](https://www.deque.com/blog/en-301-549-v4-1-1-is-final-what-changed-what-it-means-and-what-you-should-do/)). Public-sector and regulated buyers will ask for a conformance report (VPAT or ACR). `eslint-plugin-jsx-a11y` is already in the toolchain. Add automated checks with [axe-core](https://github.com/dequelabs/axe-core) (MPL-2.0) in Vitest or Playwright, and one manual screen-reader pass per release.

## 8. Motion

**When it helps.** Motion helps with feedback, state change, spatial continuity (list to detail, drawer from its trigger) and chart transitions between states. The sources are [NN/g, purpose](https://www.nngroup.com/articles/animation-purpose-ux/) and [Heer and Robertson](https://idl.uw.edu/papers/animated-transitions).

**When it hurts.**
- On frequent interactions ([Apple HIG](https://developer.apple.com/design/human-interface-guidelines/motion)).
- As staggered list entrances, which gave no tracking benefit ([Chevalier et al.](https://visualcognition.psych.northwestern.edu/publications/ChevalierDragicevicFranconeri2014.pdf)).
- When it makes people wait.
- When it is the only carrier of meaning.

**Tokens.** Put them in Tailwind 4 `@theme`, from Carbon's productive set ([Carbon](https://carbondesignsystem.com/elements/motion/overview/)), consistent with NN/g's 100 to 400 ms range ([NN/g, duration](https://www.nngroup.com/articles/animation-duration/)) and Material 3's standard easing ([M3 tokens](https://github.com/material-components/material-web/blob/main/tokens/versions/v0_192/_md-sys-motion.scss)):

| Token | Value | Use |
|---|---|---|
| `--duration-fast` | 70 to 110 ms | Toggle, checkbox, button press, copy confirmation |
| `--duration-moderate` | 150 to 240 ms | Disclosure open, tab panel change, toast in, row inserted or removed in the inbox |
| `--duration-slow` | 300 to 400 ms | Drawer, dialog, route and list-to-detail view transition |
| `--ease-standard` | `cubic-bezier(0.2, 0, 0.38, 0.9)` | Carbon productive standard |
| `--ease-enter` | `cubic-bezier(0, 0, 0.38, 0.9)` | Elements arriving |
| `--ease-exit` | `cubic-bezier(0.2, 0, 1, 0.9)` | Elements leaving, about 20% shorter than entering (NN/g) |

**Rules.**
- Animate only `transform` and `opacity`; never `transition: all`; animations must be interruptible ([Vercel guidelines](https://github.com/vercel-labs/web-interface-guidelines)).
- Under `prefers-reduced-motion: reduce`, replace movement with a short opacity change and keep feedback ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)). Use Tailwind's `motion-safe:` and `motion-reduce:` variants ([Tailwind](https://tailwindcss.com/docs/animation)).
- Spinners appear only after about 150 to 300 ms and stay at least 300 to 500 ms so they do not flash (Vercel guidelines). The demo's 1 to 16 s requests need progress text ("Running tests: 4 of 6"), not only a spinner.

**Library pick, in order of preference:**
1. **CSS and Tailwind 4 utilities.** 0 kB, MIT.
2. **The View Transition API through React 19.3 `<ViewTransition>` and TanStack Router's `viewTransition: true`.** 0 kB added. The API is Baseline since 2025-10-14, and the router ignores the option where the API is unsupported ([web-features](https://github.com/web-platform-dx/web-features/blob/main/features/view-transitions.yml), [React](https://react.dev/reference/react/ViewTransition), [TanStack](https://tanstack.com/router/latest/docs/framework/react/api/router/NavigateOptionsType)).
3. **Motion (MIT), only where 1 and 2 fall short.** This means layout animation (an inbox item leaving and the list closing up) and gesture-driven drawers. Use `LazyMotion` with `m` (about 4.6 kb initial, plus about 15 kb for `domAnimation`, loaded lazily). The figures are Motion's own ([docs](https://motion.dev/docs/react-reduce-bundle-size)). Do not import the full `motion` component (about 34 kb). Motion+ is a paid add-on and is not needed.
4. **AutoAnimate (MIT, about 3.2 kB gzip) is acceptable for simple lists.** GSAP is out because its license is proprietary and revocable ([GSAP license](https://gsap.com/standard-license/)). react-spring is more than a console needs.

All of these are npm packages bundled by Vite into the static SPA: no CDN, no runtime fetch.

## 9. Documentation

**Structure.** Use Diataxis ([diataxis.fr](https://diataxis.fr/)):
- **Tutorials:** "Your first assistant in the sandbox", "Install and check" (IT admin).
- **How-to guides:** connect MongoDB, hide and restore personal data, save runs as tests, approve tool calls, rotate keys, air-gap upgrade.
- **Reference:** OpenAPI, config keys, glossary, error codes.
- **Explanation:** the reference architecture and research notes, which already exist.

The six user docs in `core/docs/` (`datasources.md`, `evals.md`, `nodes.md`, `rag.md`, `sso.md`, `tools.md`) are mostly reference and how-to material mixed together, and should be split along these lines.

**Where the docs should live.**
- **Source:** keep user-doc Markdown in `core/docs/`, next to the code and versioned with the binary, so a feature PR updates its doc (docs as product: [PostHog handbook](https://posthog.com/handbook/content/docs)).
- **Site:** build it with Astro Starlight ([Starlight](https://starlight.astro.build/guides/site-search/), MIT). It should be a Starlight app inside `website/` (same Astro toolchain, themed from `website/DESIGN.md` tokens: B612, Atkinson Hyperlegible Next, Caliban blue) that takes `core/docs` at a release tag as its content source.
- **Search:** Pagefind (MIT) is Starlight's default and is a static index ([Pagefind](https://pagefind.app/)), so search works without a server or network.
- **Languages:** Starlight's i18n with RTL ([i18n](https://starlight.astro.build/guides/i18n/)) fits the French and Arabic-speaking customers in the demos.
- **Versions:** one docs version per minor release, fewer than about 10 kept ([Docusaurus policy](https://docusaurus.io/docs/versioning)).

**How the console links into them on-prem.**
- **Bundle the docs with the release.** The release pipeline builds a static docs bundle for that version and puts it in the Docker image and the air-gap bundle. The control plane serves it at `/docs/` on the same origin as the console, as GitLab does with its versioned docs image and `/help` setting ([GitLab](https://docs.gitlab.com/administration/docs_self_host/)). Offline docs are part of the product, not a support request ([Elastic's weaker model](https://www.elastic.co/guide/en/elastic-stack/8.9/air-gapped-install.html)).
- **The public site** publishes the latest version (and archives older ones) at the website's `/docs`.
- **Every "Learn more" is a relative link** (`/docs/<route>#<anchor>`), so it never leaves the instance. A `docs.base_url` setting can point at an internal mirror. Links are generated from the glossary file, so a CI check can verify that every anchor exists.
- **A Help button on every page, in the same place (WCAG 3.2.6),** opens the page's how-to guide in a side panel or a new tab.
- **Personalize code samples client-side** with the instance URL and the current tenant, as Stripe does with test keys ([Stripe](https://stripe.dev/blog/markdoc)). Nothing is sent anywhere.
- **No egress in the docs bundle:** no Algolia, no analytics, no external fonts (self-host them). Use [Grafana's list of switches](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/) as the audit checklist for hidden egress.

## 10. Guidance for coding agents (UI and UX skills)

Nothing was installed; this is a review. Read together, the published skills agree on a short list:
- design every state (empty, loading, error, dense);
- keyboard and visible focus everywhere;
- respect reduced motion;
- tokens before code;
- copy that is consistent and does not apologize.

They differ mainly in how much brand flair they push.

| Skill | License | Recommends | Verdict for Caliban's agents |
|---|---|---|---|
| [Anthropic frontend-design](https://github.com/anthropics/skills/tree/main/skills/frontend-design) | Apache-2.0 | Avoid generic generated looks, plan tokens, focus and reduced-motion floor, motion that answers user actions, consistent verbs, no apologetic errors | **Adapt.** Vendor the writing, motion and quality-floor rules into `web/AGENTS.md`. Its brand-distinctiveness push suits the website more than the console. |
| [Vercel web-interface-guidelines](https://github.com/vercel-labs/web-interface-guidelines) | MIT | Concrete interaction, accessibility, animation, form and content rules | **Adopt.** Vendor a pinned copy (with its license) into `web/` as review criteria for every console PR. |
| [Vercel agent-skills](https://github.com/vercel-labs/agent-skills) (web-design-guidelines, react-best-practices, react-view-transitions) | MIT (per skill files) | Review skill that fetches rules at run time; React performance rules; view-transition rules | **Skip** the fetching skill; **Adapt** the bundle and re-render rules and the view-transition principles, noting the stale canary advice. |
| [Impeccable](https://github.com/pbakaus/impeccable) | Apache-2.0 | Audit, critique and polish commands, detectors, PRODUCT.md and DESIGN.md, motion timing for "operate" surfaces | **Adapt.** The website already uses its PRODUCT.md schema. Use its reference text; do not run its CLI in builds (hosted API calls, opt-out telemetry). |
| [Google DESIGN.md](https://github.com/google-labs-code/design.md) | Apache-2.0 | Token spec plus linter with contrast checks | **Adopt.** Write `web/DESIGN.md` with the console's tokens, including the motion tokens above. |
| [shadcn skill](https://github.com/shadcn-ui/ui/tree/main/skills/shadcn) | MIT | Semantic tokens, accessible fields, Empty and Skeleton components | **Skip** (network at load, different component stack). |
| [UI UX Pro Max](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill) | MIT | Style and palette catalogs, Google Fonts | **Skip.** |
| [web-artifacts-builder](https://github.com/anthropics/skills/tree/main/skills/web-artifacts-builder) | Apache-2.0 | Single-file artifacts on React 18 and Tailwind 3 | **Skip.** |

**Recommended practice.**
- **Do not install third-party skills.** Write one in-repo `web/AGENTS.md` (or a project skill under the repo, never in `~/.claude`) that combines four sources:
  - the vendored Vercel rules (MIT, keep the notice);
  - the relevant frontend-design rules (Apache-2.0, keep attribution);
  - this note's glossary and motion tokens;
  - the house rules: no em dashes in UI copy, zero egress, no CDNs.
- **Name this note in every UI agent brief**, as the research-grounded workflow requires.

---

## Risks and failure modes

1. **Defaults that hide a safety-relevant choice.** "Hide and restore" vs "hide permanently" changes what callers see. Carbon's rule applies: never hide what the task needs. Mitigation: persona-specific basics, and a review step that shows every defaulted safety setting.
2. **"Go live" collapsing three steps could hide a failing gate.** Mitigation: the one button runs the steps visibly and stops with a plain reason ("2 of 6 tests failed; the release bar is 80%"). Bar changes below the template default require a reason in the audit log. The demo's fintech bar of 0.6 is the cautionary case.
3. **Natural-language spec drafts that look right and are wrong.** Mitigation: draft-only, the form review step and the eval gate. Never auto-publish.
4. **Renaming drift.** If the API, SDKs and docs keep the old terms while the UI changes, users have to translate. Mitigation: the glossary file maps UI terms to API names, and docs show both ("Assistant (API: node)").
5. **Sample tenant in production.** Mitigation: a clear "Sandbox" badge, excluded from billing and audit exports, with one-click removal and a deploy flag to skip it.
6. **Docs bundle size and freshness.** Mitigation: Pagefind's chunked index stays small; docs are versioned with the release, so the bundle cannot be stale relative to the binary.
7. **Motion regressions on low-power VDI sessions,** which are common in banks and hospitals. Mitigation: transform and opacity only, the reduced-motion path, and a manual "reduce motion" switch in user settings that applies even when the OS preference is not exposed through VDI.

## Open questions

1. **Names.** Do "Assistant", "Data map", "Query plan" and "Release bar" test well with the three buyer types (hospital, law firm, bank)? Five users per persona, task: "make an assistant that answers from this database". Measure time to first answer and count the questions they ask.
2. **Activation metric.** Define "first answer" (first successful run from a published assistant, through a key) and measure it locally, from the audit log, with no telemetry. What target: under 15 minutes from login on the sandbox, under 60 minutes on the customer's own data?
3. **Where does the ontology step go for a domain owner?** Can the wizard accept only what the template's assistant uses, and leave the rest of the data map unreviewed and unusable until someone reviews it?
4. **"Describe your assistant" on a small on-prem model.** Is a 27B model good enough to draft specs, or does this need a dedicated small fine-tune trained on the spec schema? Evaluate it on the four demo scenarios.
5. **Docs location in git.** Should the Starlight app live in `website/` (shared brand) or `core/` (one repo per release artifact)? This note recommends source in `core/docs` and the app in `website/`. The release pipeline then needs both checkouts.
6. **React Aria vs Base UI.** React Aria has broader internationalization (Arabic, RTL, calendars). Base UI is lighter and closer to Radix. Prototype the palette and the dialog in both before committing.
7. **Accessibility conformance report.** Who signs it, and against EN 301 549 V3.2.1 or V4.1.1, given that V4.1.1 is not yet cited in the Official Journal?

---

## What Caliban should do (prioritized)

Persona tags: **[Admin]** IT admin who installs once; **[Owner]** domain owner who builds assistants; **[Both]**; **[Agents]** coding agents and maintainers. Effort: S (days), M (one to two weeks), L (several weeks).

1. **Glossary and renames.** One glossary file (label, definition, doc anchor, aliases) extending `lib/labels.ts`. Rename the on-screen jargon (section 5), add toggletips, and lint banned terms in basic UI copy. **[Both] M**
2. **"Build an assistant" in 4 steps.** Template or description, data, try it, share it. Intent, PII, caps and gate are defaulted, and the step ends on a review screen showing the generated spec. **[Owner] L**
3. **Bundled sandbox tenant.** Offline fixtures, one published assistant with tests, a reset action, a "Sandbox" badge and a deploy flag to skip it. **[Owner] M**
4. **Templates from the four demo scenarios.** Specs, golden cases and example requests packaged as starting points. **[Owner] S**
5. **Form-first spec editor with a JSON tab.** Keep `node-editor.tsx` as the JSON view. Specs the form cannot show get the Cloudflare-style "JSON only" notice. **[Owner] L**
6. **Role-based navigation and two-level disclosure.** A domain-owner role sees 6 sections. Every settings card splits into basic and "Advanced", and every default is explained. **[Both] M**
7. **Headless first boot and the Install checklist.** Deploy config creates the first model, tenant and admin. The Overview checklist is derived from live state, with "Test connection" checks. **[Admin] M**
8. **"Save as test" on every run, plus test-the-judge-on-one-case.** Golden sets grow from real runs; misroutes become routing tests in one click. **[Owner] M**
9. **Close the console gaps.** Edit request-type-to-assistant routes and the tenant default route in the console, not only through the API. **[Admin] S**
10. **Docs shipped with the binary.** Starlight and Pagefind, sourced from `core/docs` and restructured with Diataxis. Served at `/docs/<version>/` by the control plane and published on the website. Every "Learn more" deep-links to an anchor checked in CI. **[Both] L**
11. **Empty states and errors rewritten to the rules.** One action per empty state. Control-plane errors map to fields with a fix. Status messages use `role="status"`. **[Both] S**
12. **Ontology review as a batch plus a keyboard queue.** Accept high-confidence entries used by the assistant in one step; review the rest with A, R and E keys. **[Owner] M**
13. **Clearer eval results and approval inbox.** Eval results show failures first, with regression-first version compare. The inbox offers approve, edit and reject with a reason, shows expiry, has an optional no-self-approval rule and supports keyboard review. **[Owner] M**
14. **Accessibility and palette foundation.** Move dialogs, tabs, drawer and menus onto React Aria or Base UI. Add a Cmd/Ctrl+K palette with glossary aliases. Run axe-core in CI and audit against the WCAG 2.2 AA checklist in section 7. **[Both] M**
15. **Motion tokens and agent guidance.** Carbon productive tokens in `@theme`, reduced-motion handling, and view transitions on route and list-to-detail, with Motion `LazyMotion` only where needed. Add `web/DESIGN.md` and `web/AGENTS.md` with the vendored Vercel guidelines (MIT) and frontend-design rules (Apache-2.0), with no runtime-fetching skills. **[Agents] S**
