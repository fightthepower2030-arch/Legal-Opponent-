# CLAUDE.md

Guidance for Claude Code (and any other contributor) working in this repository.

## What the project does

Legal-Opponent is an AI tool for **litigants in person** (people who conduct
civil proceedings without a solicitor or barrister) to research the opposing
party in their case. The README's one-line description is currently
unfinished ("allows litigants in person to search their Opponents"); its exact
scope (company records, previous litigation, published judgments, and so on)
has not yet been written down. Confirm scope with the maintainer before
building features based on an assumed scope.

## Current state of the repository

As of this file's creation the repository contains only:

| Path         | Contents                                         |
|--------------|--------------------------------------------------|
| `README.md`  | Project title and a one-line (unfinished) description |
| `LICENSE`    | GNU General Public License v2                    |
| `CLAUDE.md`  | This file                                        |
| `.claude/agents/` | Sub-agent definitions (see below)           |

There is **no application code, no package manifest, no tests and no CI** yet.

## Tech stack

**Not yet chosen.** Nothing in the repository commits to a language,
framework, database or hosting platform. When a stack is chosen, update this
section with: language and version, framework, package manager, database,
LLM provider/SDK, and hosting target.

## How to run and test

**No run or test commands exist yet.** When the first code lands, record here:

- install command (e.g. dependency install)
- run / dev-server command
- test command (and how to run a single test)
- lint / format / typecheck commands

Do not invent commands in this file; only document ones that actually work in
the repository.

## Folder structure

Only the root files listed above exist. Record the layout here once the first
code is added (for example `src/`, `tests/`, `docs/`).

## Coding conventions

No code exists yet, so there are no observed conventions. Until the stack is
chosen, follow these defaults:

- Keep changes small and focused; one concern per commit.
- Every new feature ships with tests.
- Configuration and secrets come from environment variables (with a committed
  `.env.example` listing names only, never values).
- Licence: the project is GPL-2.0. Any dependency added must be licence-compatible
  with GPL-2.0 (note that Apache-2.0 is generally regarded as incompatible with
  GPL-2.0-only); check before adding.

## Rules (mandatory)

These rules override convenience. If a task would breach one, stop and ask.

1. **England and Wales only.** The app serves the jurisdiction of England and
   Wales. Do not add features, content, data sources, court lists or legal
   material for Scotland, Northern Ireland or any other jurisdiction. Where a
   source covers several UK jurisdictions (e.g. legislation.gov.uk, Companies
   House), filter or label so users are only given material applicable in
   England and Wales, and make the extent of any statute clear.
2. **Legal citations must come from a verified source, never from memory.**
   Any case name, neutral citation, law-report reference, statute, section or
   procedural rule that the app outputs must be retrieved from, and linked to,
   an authoritative source at runtime (for example legislation.gov.uk, The
   National Archives' Find Case Law, or the Civil Procedure Rules on
   justice.gov.uk). An LLM must never be allowed to generate a citation from
   its own training data. If a citation cannot be verified, the app must say
   so rather than output it. The same applies to Claude when writing code,
   fixtures, prompts or documentation: do not write real-looking legal
   citations from memory; use clearly fictitious placeholders in tests or
   fetch and verify the real source.
3. **Never commit secrets or API keys.** No keys, tokens, passwords,
   connection strings or personal data in code, tests, fixtures, commit
   messages or documentation. Use environment variables and keep `.env*`
   files (other than `.env.example`) out of git.
4. **Ask before changing anything touching payments or user data.** Any change
   to payment flows, billing, pricing, user accounts, authentication,
   personal data storage, retention, or data about opponents (who are third
   parties and data subjects in their own right) requires explicit approval
   from the maintainer before it is made.

## Sub-agents

Defined in `.claude/agents/`:

| Agent        | Role                                                         |
|--------------|--------------------------------------------------------------|
| `builder`    | Implements features and fixes                                |
| `reviewer`   | Reviews diffs with fresh eyes; never reviews its own work and never edits |
| `tester`     | Writes and runs tests                                        |
| `researcher` | Reads documentation and evaluates libraries/data sources; read-only |

Typical flow: `researcher` → `builder` → `tester` → `reviewer`. The reviewer
must be a separate invocation from whichever agent wrote the change.

## Git

- Never commit directly to `main`; work on a feature branch and open a PR.
