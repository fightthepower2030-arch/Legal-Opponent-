# CLAUDE.md

Guidance for Claude Code (and any other contributor) working on this project.

## What the project does

**AI LAW & MORE SOCIAL** (ailawandmoresocial.com) is a platform for
**litigants in person** (people conducting proceedings in England and Wales
without a solicitor or barrister). It combines a social network for
litigants with case-management and legal-research tools. It is currently in
**testing**, not general release.

Main features:

- Social feed, communities, direct messaging, follows, blocks, bookmarks.
- **Case Rooms**: shared case workspaces with parties, events, issues,
  propositions, evidence and authorities.
- **Legal Check**: checks a legal proposition with an AI model and web search
  restricted to official sources, then verifies every returned citation
  against the official record (see "Citation verification" below).
- **Claim-pack extraction**: AI extraction of reviewable fields from uploaded
  possession claim packs.
- **OPR engine**: eligibility, deadline and court-centre logic for
  possession claims (`server/opr.ts`).
- Secure document storage, data export, account deletion, moderation,
  appeals and professional verification.

**Access during testing is invite-only.** Every endpoint calls
`requireUser` (`server/profile.ts`), which refuses any signed-in user whose
email is not an active row in the **Tester Allowlist** table
(`server/testerGate.ts`, `server/testerAccess.ts`). Uninvited users see an
"Invite-only testing" screen. Testers are added or removed in the Zite
database, not in code. Removing the gate or changing who can sign in falls
under rule 4.

## Where the code lives

The application code is **not in this GitHub repository**. It lives in a
**Zite** workspace (Zite is a hosted app builder; the workspace is its own git
repository):

- Workspace: "AI LAW & MORE SOCIAL" (id `c849de3ab3e6804d`)
- Apps:
  - `apps/ai-law-more-social`: the public site (Zite app id `6r4oyh1fjg`)
  - `apps/ai-law-more-control`: the staff/admin console for moderation, appeals,
    professional review and staff roles (Zite app id `wgzexrsnpu`)

Work on the app through the Zite tools (sandbox → edit → `check_app` →
`commit`). Never `git commit`/`git push` inside the Zite sandbox; use its
`commit` tool. **Committing does not publish**: the live site only changes
on `publish_app`, which requires the maintainer's explicit go-ahead.

This GitHub repository holds the project's Claude Code configuration
(this file and `.claude/agents/`), plus the README and licence.

## Tech stack

- **ZiteJS** 0.9.x (pinned in the root `package.json`): React + TypeScript +
  Vite front end, with typed backend endpoints.
- **UI**: Tailwind CSS, shadcn/ui (`packages/components`), lucide-react.
- **Database**: Zite's hosted Postgres, accessed via `import { zite } from
  "zitejs/db"` (typed client in `.zite/db.ts`; raw SQL via `zite.sql()` with
  `$n` parameters only).
- **Integrations** (configured in the root `zite.config.json`):
  - `openai`: Legal Check and claim-pack extraction
  - `googleDrive`: secure document storage
- **Tests**: vitest.

## How to run and test (inside the Zite sandbox, from `/workspace`)

| Task | Command |
|---|---|
| Unit tests | `npx vitest run` (or `npm test`) |
| One test file | `npx vitest run apps/ai-law-more-social/src/server/<file>.test.ts` |
| Typecheck and endpoint bundling | Zite `check_app` tool (or `npm run typecheck`) |
| Regenerate the API typings after adding or renaming endpoints | `npx zitejs generate` (run from `/workspace`) |
| Preview | Zite editor, via the `editorUrl` returned by `commit` |
| Runtime debugging | Zite `get_logs`; `run_one_off_script` for probing the runtime |

Note: vitest only picks up `*.test.ts`. The files `opr.selftest.ts` and
`fileValidation.selftest.ts` do **not** run in `npm test`.

## Folder structure (public app)

```
apps/ai-law-more-social/
  src/App.tsx              root UI component
  src/components/          feature panels (Case Intelligence, Digital Twin, OPR, Claim Pack, public legal pages)
  src/api/*.ts             one backend endpoint per file (createEndpoint + zod schemas)
  src/server/*.ts          shared server logic (auth/profile, case access, OPR, moderation,
                           rate limits, drive storage, citation verification) and tests
  zite.config.json         per-app config (accessMode, integration settings)
packages/components/       shared shadcn/ui design system
.zite/                     generated SDKs; never edit
```

## Coding conventions (observed)

- One endpoint per file in `src/api/`, default-exporting `createEndpoint`
  with zod `inputSchema` and `outputSchema`.
- Every endpoint that touches user data starts with `requireUser(context)` or
  `getOrCreateProfile(context)`; Case Room endpoints call
  `requireCaseRoomAccess(context, roomId)`.
- AI features enforce per-user hourly and daily limits via the `AiUsage`
  table and record Started, Completed and Failed outcomes.
- User-supplied text sent to a model is fenced as untrusted, and prompts tell
  the model to ignore embedded instructions.
- Shared logic belongs in `src/server/`, with a colocated `*.test.ts`.
- External services are mocked in tests; no live network calls in `npm test`.
- Compact style: two-space indentation and terse expressions, matching the
  existing files.

## Rules (mandatory)

These rules override convenience. If a task would breach one, stop and ask.

1. **England and Wales only.** Do not add features, content, data sources or
   legal material for Scotland, Northern Ireland or any other jurisdiction.
   Where a source covers several jurisdictions, check its extent and exclude
   or label material that does not apply in England and Wales.
2. **Legal citations must come from a verified source, never from memory.**
   This applies both to what the app outputs and to anything Claude writes
   (code, prompts, fixtures, docs):
   - A case or legislation reference the app shows as authority must have
     been matched against the official record at runtime
     (`server/citationVerification.ts`).
   - Anything that cannot be matched is withheld and labelled unverified.
     It must never be shown as authority.
   - Checking the URL host alone is **not** verification.
   - Tests use clearly fictitious parties, or real records fetched from the
     official source, never citations recalled from memory.
3. **Never commit secrets or API keys.** Integration credentials are injected
   by Zite (`ZITE_*` env vars). User-supplied secrets are declared under
   `envVars` in the app's `zite.config.json`, and the maintainer sets their
   values in the Zite editor. `.env.local` stays gitignored.
4. **Ask before changing anything touching payments or user data.** That
   covers payments and billing, accounts and authentication, personal-data
   storage, export and erasure, retention, access modes, and data about
   opponents and other case parties (third parties with their own data
   protection rights). Get explicit approval from the maintainer first.
5. **Never publish without approval.** Commit to preview; only `publish_app`
   when the maintainer says so.

## Citation verification (how it works)

`apps/ai-law-more-social/src/server/citationVerification.ts`:

- **Cases**:
  - Parses the neutral citation (UKSC, UKPC, EWCA, EWHC, EWFC, EWCOP, UKUT,
    EAT) and builds the canonical Find Case Law address. The model's URL is
    ignored.
  - Fetches `<address>/data.xml` and requires the official citation, year,
    court family and case name to match.
  - Citations that can't be checked automatically, such as House of Lords and
    pre-2001 cases, are unverified and must be checked manually.
- **Legislation**:
  - legislation.gov.uk only. Requests need a user-agent header or the site
    refuses them.
  - Reduces any link (PDF copy, `/contents`, point-in-time) to the
    canonical address and builds the provision's own address when the link
    is to the whole Act.
  - Acts that the model lists as cases (with no citation) are checked as
    legislation.
  - Fetches `data.xml` and requires the title and provision to match, and
    the extent to include E or W.
- **Prose**: any citation in the written analysis that wasn't verified in
  the same run (and all law-report citations) is replaced with
  "[unverified citation removed]". Citations the user typed are kept.
  Withheld cases mentioned by name only are marked "[unverified]".
- **Display**: verified cases appear under "CASE LAW (VERIFIED)", with a
  court and precedent label derived from the citation
  (`src/courtStanding.ts`), and each record is listed once.
- **Fails closed** on network errors, timeouts, non-200 responses, unreadable
  records or any mismatch.
- **Not yet checked**:
  - Whether a source actually supports the proposition it is cited for.
  - A case's later history (appeals, overruling). Every verified case below
    the Supreme Court carries a warning to confirm it was not overturned.
  - House of Lords and pre-2001 judgments, which Find Case Law does not
    hold. BAILII cannot be used as an automated source: it serves a bot
    challenge, and that must not be circumvented.
  The UI says so.

## Sub-agents

Defined in `.claude/agents/`:

| Agent | Role |
|---|---|
| `builder` | Implements features and fixes |
| `reviewer` | Reviews diffs with fresh eyes; never reviews its own work and never edits |
| `tester` | Writes and runs tests |
| `researcher` | Reads documentation and evaluates libraries and data sources; read-only |

Typical flow: `researcher` → `builder` → `tester` → `reviewer`. The reviewer
must be a separate invocation from whichever agent wrote the change.

## Git (this repository)

Never commit directly to `main`; work on a feature branch and open a PR.
