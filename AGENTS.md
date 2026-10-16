# Maintaining this repo

This repo ships two Agent Skills (`brand-setup`, `content-creator`) in the portable
`SKILL.md` format, meant to work in Claude Code, Codex and any other agent that loads
skills. These are the rules for editing them; see `README.md` for the user-facing pitch
and usage. `AGENTS.md` is the canonical copy of these rules — `CLAUDE.md` only imports
it (`@AGENTS.md`), so edit this file, not that one.

## Source of truth

`skills/` is the only copy of both skills in this repo — there is no `.claude/skills/`
or `.agents/skills/` here, this repo is source only. Using the skills means installing
them into a project (`npx skills add allangrds/influencer-skill` or `claude plugin
install`), which is what actually puts them somewhere an agent looks for skills.

## Agent-agnostic skills

The skills must not assume a specific agent:

- Reference bundled files by paths relative to the skill's own directory (e.g.
  `templates/page-video.html`), never by an install location like
  `.claude/skills/...` — that path differs per agent and per install method.
- Describe capabilities, not tool names. When a step benefits from an agent-specific
  tool (`AskUserQuestion`, `WebSearch`, subagents), name it only as an example and give
  a fallback for agents that don't have it.

## Valid network slugs

`article`, `youtube`, `instagram`, `tiktok`. Adding a new one means keeping these in sync:

- `skills/content-creator/SKILL.md` — the research + generation + revision procedure
- `skills/content-creator/templates/` — the HTML/Tailwind page skeleton for that network
- the `index.html` generation logic described below

## Generated user data — never commit

Everything from `BRANDING.md` down (the six brand-setup config files, `content/`,
`index.html`) is real creator data produced by *using* the skills, not repo source. It's
listed in `.gitignore`; keep it that way. If you add a new output file to either skill,
add it to `.gitignore` too.

## Content-generation invariants

- Each network's page is generated in isolation (its own subagent, in parallel, where
  the agent supports it; otherwise one at a time) and is fully self-contained (Tailwind
  via CDN, no build step) — this is what keeps later revisions cheap, since editing one network's page never requires reloading research or touching
  the other networks' content.
- `index.html` is never rebuilt from scratch. Each `content-creator` run reads it,
  updates only the current topic's section, and leaves the rest untouched.
- `instagram` is a single network with three interchangeable formats — `reel`, `post`,
  `carousel` — chosen per topic, not per brand. It's still one output file
  (`pages/instagram.html`) regardless of format; the chosen format is recorded in an
  `<!-- instagram-format: ... -->` comment in that file so revisions don't need to
  re-ask. `templates/page-instagram-post.html` and `templates/page-instagram-carousel.html`
  must stay in sync with `SKILL.md`'s format branch (same spirit as the network-slug sync
  rule above).
- Sources show up in two layers. **For the creator:** every research reference used in
  a page gets an amber source box right after it (what the source says, with the
  verbatim excerpt; caveats when needed; the full reference of every kind of source,
  link included), plus a legend at the top of each section. It's styled apart from the
  content and hidden when printed (`.source-ref`). **For the public:** the content
  itself cites formally — in-text citations and a references section in `article`,
  on-screen citation cues in video scripts, the full list in the YouTube description,
  and a pointer to it in the other captions. Every `templates/page-*.html`, including
  one for any new network, must carry the `.source-ref` print rule and the legend +
  box snippet comment, and the research step must keep writing source IDs, verbatim
  excerpts, a `Reference:` line and a `Citation:` line to `RESEARCH_SOURCE.md` so the
  pages have something to show.
- The citation style (ABNT, APA 7, …) lives in the `## Citation style` section of
  `COMMUNICATION_STYLE.md`. `brand-setup` asks for it and writes it there;
  `content-creator` reads it, with a default by language when it's missing. Keep the
  question, the template section and that fallback in sync.

## SKILL.md frontmatter conventions

Both skills use `user-invocable: true` and a `description` written as trigger phrases
(the actual sentences a user would say), not a generic summary — follow the existing
pattern in `skills/brand-setup/SKILL.md` and `skills/content-creator/SKILL.md` when
editing either. Keep `description` in sync with the real invocation phrases documented
in `README.md`'s Usage section.

`user-invocable` and `argument-hint` are Claude Code extensions, not part of the
[Agent Skills spec](https://agentskills.io/specification) — other agents ignore them,
and `skills-ref validate` flags them; that's a known, accepted trade-off. Any other new
field should come from the spec (`license`, `compatibility`, `metadata`,
`allowed-tools`) unless it's a deliberate Claude-only addition.

## Distribution

Two independent install paths, both driven by this same `skills/` tree — keep them in
sync when adding/removing a skill:

- `npx skills add allangrds/influencer-skill` (skills.sh, any supported agent incl. Claude
  Code and Codex) — see `README.md`'s Publishing section.
- `claude plugin install` (Claude Code only) via `.claude-plugin/plugin.json` / `marketplace.json` — the
  `skills` array in `plugin.json` must list every skill under `skills/`.
- The plugin `name` is `influencer` (in both `plugin.json` and `marketplace.json` — keep
  them equal). Claude Code namespaces plugin skills as `<plugin>:<skill>`, so this is what
  makes them `/influencer:brand-setup` and `/influencer:content-creator`. Don't add the
  prefix to skill folders or `name:` frontmatter — it would double up as
  `influencer:influencer-…`, and dots aren't valid in skill names for skills.sh.
