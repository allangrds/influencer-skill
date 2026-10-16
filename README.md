# influencer-skill

Agent Skills that turn a creator's brand profile + a topic into on-brand,
researched content (video scripts, captions/hashtags, or an article) for every
network the creator uses. They use the portable `SKILL.md` format, so they work in
Claude Code, Codex and other agents that support skills, and are
installable/publishable via [skills.sh](https://skills.sh).

## How it works

Two skills, used in order:

1. **`brand-setup`** — one-time (or occasional) interview that captures the creator's
   brand, tone, main/secondary networks, inspirations, and desired content structure.
   Writes six Markdown files at the project root.
2. **`content-creator`** — given a topic, researches it, then generates one HTML page
   per configured network: a shot-by-shot video script + caption/hashtags for `youtube`
   and `tiktok`, an article body for `article`, and for `instagram` whichever format you
   pick — Reel (shot-by-shot script), regular feed Post (a visual concept), or Carousel
   (numbered slides) — plus a caption/hashtags block. Every fact or quote taken from
   the research is followed by a source box for the creator (what the source says, the
   original excerpt, the full reference and link), styled apart from the script so it's
   clearly not part of what gets recorded or published. The content itself cites its
   sources formally, in the citation style chosen in `brand-setup` (e.g. ABNT, APA):
   in-text citations and a references list in the article, on-screen citations in the
   videos, the full list in the YouTube description. Keeps a root `index.html` of
   everything produced.

## Usage

```
"configure my brand"          → runs brand-setup
"change my citation style"    → brand-setup, asks only that
"create content about <topic>" → runs content-creator
"create an instagram carousel about <topic>" → content-creator, asks/uses the instagram format
"change the hook on the YouTube script for topic 001" → content-creator's revision flow
```

Or invoke them directly. Installed as a Claude Code plugin, the skills are namespaced
under the plugin name `influencer`:

```
/influencer:brand-setup
/influencer:content-creator <topic>
```

Installed via `npx skills add`, they're plain `/brand-setup` and `/content-creator <topic>`
in Claude Code. In Codex and other agents, invoke them however that agent exposes
skills (e.g. `$content-creator` in Codex), or just ask in plain language.

Run `brand-setup` first — `content-creator` refuses to run until the brand files exist.
Re-running `brand-setup` later only asks about what's missing or what you explicitly
want to change; it won't repeat the full interview.

## Project structure

```
influencer-skill/
├─ skills/                        # the only copy of both skills — this repo is source only
│  ├─ brand-setup/
│  │  ├─ SKILL.md                 # interview procedure
│  │  └─ templates/                # blank templates for the 6 root config files
│  └─ content-creator/
│     ├─ SKILL.md                 # research + generation + revision procedure
│     └─ templates/                # HTML/Tailwind page + index skeletons (+ source box snippet)
│        ├─ page-video.html        # youtube, tiktok, instagram reels
│        ├─ page-instagram-post.html
│        ├─ page-instagram-carousel.html
│        ├─ page-article.html
│        └─ index.html
│
├─ .claude-plugin/                # plugin manifests for `claude plugin install`
├─ skills.sh.json                 # groups the skills on the skills.sh marketplace page
├─ AGENTS.md                      # maintenance rules for any agent (canonical)
├─ CLAUDE.md                      # just `@AGENTS.md`, so Claude Code loads the same rules
├─ .gitignore                     # keeps generated/personal files out of git
│
├─ BRANDING.md                    # created by brand-setup: name, what they do, audience
├─ COMMUNICATION_STYLE.md         # created by brand-setup: tone of voice, citation style
├─ MAIN_INFLUENCE_ZONE.md         # created by brand-setup: the one primary network
├─ SECONDARY_INFLUENCE_ZONE.md    # created by brand-setup: zero or more secondary networks
├─ INSPIRATIONS.md                # created by brand-setup: reference creators/brands
├─ STRUCTURES.md                  # created by brand-setup: content structure per network
├─ index.html                     # created/updated by content-creator: index of all topics
│
└─ content/
   └─ <NNN>-<topic-slug>/         # one folder per topic, NNN = 001, 002, ...
      ├─ RESEARCH_SOURCE.md       # sources (S1, S2, …): name, title, URL, excerpts, full reference
      ├─ RESEARCH.md              # synthesized research, each fact tagged with its [S#]
      └─ pages/
         ├─ youtube.html          # one file per configured network
         ├─ instagram.html
         └─ ...
```

This repo holds no installed copy of the skills, only the `skills/` source — same
layout as other public skill collections. To actually use them, install into a project
with `npx skills add allangrds/influencer-skill` or `claude plugin install` (see below); that's what
puts them somewhere your agent looks for skills (e.g. a project's `.claude/skills/` or
`.agents/skills/`, or a Claude Code plugin's skill set).

Everything from `BRANDING.md` down is what a project that has the skills installed ends
up with after actually *using* them. This repo's `.gitignore` keeps that pattern out in
case you ever run the skills here directly — it holds a real creator's brand and
content, so it must never end up in a public copy of this repo.

Valid network slugs (used both in the config files and as filenames): `article`,
`youtube`, `instagram`, `tiktok`.

See `AGENTS.md` for the canonical maintenance rules (source of truth, gitignore
invariants, frontmatter conventions) when editing either skill.

## Design notes

- Each network's page is generated in isolation (its own subagent, in parallel, where
  the agent supports it; otherwise one at a time) and is fully self-contained (Tailwind via CDN, no build step). That's what makes later revisions
  cheap: editing one network's page never requires reloading research or the other
  networks' content.
- `index.html` is never rebuilt from scratch — each `content-creator` run reads it,
  updates only the current topic's section, and leaves the rest untouched.
- Sources show up in two layers. Every fact, number or quote a page takes from the
  research is followed by a source box for the creator: what the source says (with the
  verbatim excerpt), caveats when needed, and the full reference with its link — for
  every kind of source, not only books. It isn't part of the content, so it's styled
  apart from the script (amber box, "not part of the script" label, a legend at the
  top) and hidden when the page is printed. The published content cites formally, in
  the brand's citation style: in-text citations and a references section in the
  article, on-screen citation cues in video scripts, the full list in the YouTube
  description, and a pointer to it in the other captions. This works because the
  research step stores each source with an ID, verbatim excerpts, its full reference
  and its in-text citation in `RESEARCH_SOURCE.md`, and tags each fact in `RESEARCH.md`
  with that ID. The page agents follow the tag, so no new research is needed.

## License

[MIT](LICENSE) © 2026 Allan Ramos. Free to use, including commercially, as long as the copyright notice and license text are kept.
