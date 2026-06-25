# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Documentation site for **Vero Monitor Service (VMS)** built on **Mintlify**. Content only — there is no `package.json`, build script, test suite, or CI. The entire site is `.mdx` pages plus assets, configured by `docs.json`. Mintlify renders and hosts it.

Scope is operator-facing and limited to three workflows: set up the VMS master control plane, roll out VMS agent collectors, and troubleshoot collectors/dashboards/alerts/probes. Do not add trading-platform integration content (see `skill.md`).

## Bilingual structure — keep en/ and vi/ in sync

Content lives in two parallel trees, `en/` (English) and `vi/` (Tiếng Việt), with **identical file sets**. Every page exists in both languages.

- A change to any `en/` page must be mirrored in its `vi/` counterpart, and vice versa. Recent history shows pages are edited in pairs ("vi and en kept in sync"); breaking parity is a defect.
- Translate the prose, but keep the technical content identical across languages — commands, paths, config field names, metric names, and table structure must match exactly.

## Navigation and adding pages

`docs.json` is the Mintlify config: theme, colors, fonts, SEO, and the `navigation` tree. Navigation is defined **separately per language** under `navigation.languages[]` (one block for `en`, one for `vi`), each with its own `groups`.

A page only appears in the site if it is listed in `docs.json`. To add a page you must do all three: create `en/<page>.mdx`, create `vi/<page>.mdx`, and add the path to **both** language groups in `docs.json`. (Note: `en/probes/container-mon.mdx` and `en/probes/k8s-mon.mdx` exist on disk but are absent from navigation, so they are not published — match an existing pattern before relying on either.)

Page paths in `docs.json` omit the `.mdx` extension and are relative to repo root (e.g. `en/probes/ping`).

## Page conventions

Every page starts with YAML frontmatter: `title`, `description`, and optionally `mode` (`'wide'` / `'custom'`). Pages use Mintlify components — `<Card>`/`<CardGroup>`, `<Steps>`/`<Step>`, `<Warning>`/`<Info>`/`<Note>` — and inline JSX (the progress bars in setup pages, video heroes).

**Probe pages** (`en/probes/*` and `vi/probes/*`) follow a fixed template; reuse it for any new probe:
1. `## Description` — what the probe does and platform/adapter requirements
2. `## Config fields` — a table: `Field | Type | Required | Default | Description`
3. `## Metrics` — a table: `Metric | Type | Unit | Labels | Condition`
4. `## Example config`
5. `## Notes`

## Docs must match the shipping agent

The bulk of recent work is correcting docs to match what the agent actually ships. Config field names, defaults, metric names/labels (e.g. `vms.host.cpu.usage_percent`), CLI invocations (`agent.sh` / `agent.ps1`), file paths, and service models (Linux `systemctl --user`, Windows Scheduled Task) must reflect the real agent — not aspirational or outdated behavior. When unsure whether a documented command/field is real, flag it rather than inventing one.

## Writing style

Plain, operator-focused, concise. **No marketing language.** Write the simplest sentence that conveys the fact; cut adjectives, hype, and filler. Prefer short declarative sentences, tables, and step lists over prose. State what something does and how to use it — nothing more.

## Local preview

This is a standard Mintlify project (config is `docs.json`). Preview locally with the Mintlify CLI from the repo root:

```bash
npm i -g mint   # one-time
mint dev        # serves a live local preview
```

There is no lint or test step; correctness is verified by previewing the rendered site and by keeping en/vi parity and `docs.json` navigation in sync.
