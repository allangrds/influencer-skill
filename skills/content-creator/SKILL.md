---
name: content-creator
description: Research a topic and generate video scripts plus social captions (or an article) across the creator's configured networks, using the brand profile set up by brand-setup. Use when the user asks to create content, write a script, draft a post, or research a topic for their channel — e.g. "create content about X", "write me a YouTube script on X", "make a post about X". Also use for follow-up edits to already-generated content, e.g. "change the hook on the Instagram script for topic 001".
---

# Content Creator

Turns a topic into researched, on-brand content for every network the creator has
configured, using `brand-setup`'s output files as the source of truth. Produces one
self-contained HTML+Tailwind page per network plus a root `index.html` that indexes
every topic ever produced.

## Preconditions

Before doing anything, check that these exist at the project root and have real
content: `BRANDING.md`, `COMMUNICATION_STYLE.md`, `MAIN_INFLUENCE_ZONE.md`,
`STRUCTURES.md`. If any is missing or empty, tell the user to run the `brand-setup`
skill first and stop — do not guess brand info.

## File layout this skill manages

```
content/<NNN>-<topic-slug>/
  RESEARCH_SOURCE.md      # list of sources found (title + URL + 1-line note)
  RESEARCH.md             # synthesized findings on the topic
  pages/
    <network>.html        # one per configured network, e.g. youtube.html, article.html
index.html                # root index of every topic, updated (never fully rebuilt) each run
```

`<NNN>` is a zero-padded sequence number (`001`, `002`, ...). `<topic-slug>` is the
topic kebab-cased (lowercase, spaces/punctuation → `-`).

## Procedure — new topic

1. **Determine the topic and slug.** List `content/` to find the highest existing
   `<NNN>` prefix; the new one is that + 1 (use `001` if `content/` doesn't exist yet).
   Create `content/<NNN>-<slug>/` and `content/<NNN>-<slug>/pages/`.

2. **Determine target networks.** Read `MAIN_INFLUENCE_ZONE.md` and
   `SECONDARY_INFLUENCE_ZONE.md`. By default generate for all configured networks. If
   the user explicitly asked for only some networks in their request, honor that
   instead.

3. **Research (one subagent).** Launch a single fresh agent (general-purpose, needs
   WebSearch/WebFetch/Write) with a self-contained prompt along these lines:

   > Research the topic "`<topic>`" for a content creator. Find credible, current
   > sources (articles, official docs, reputable news, studies — avoid low-quality
   > SEO content). Write `content/<NNN>-<slug>/RESEARCH_SOURCE.md` as a bullet list,
   > each bullet: title, URL, and a one-line note on what it's useful for. Then write
   > `content/<NNN>-<slug>/RESEARCH.md`: a synthesized write-up of the topic covering
   > key facts, useful angles/hooks for content, and anything surprising or
   > counter-intuitive worth highlighting. Do not write any HTML or scripts — only
   > these two Markdown files.

   Wait for it to finish before continuing. This keeps raw search results out of your
   own context — you only need to know it succeeded.

4. **Generate one page per network (parallel subagents).** In a single message, launch
   one fresh agent per target network (all in parallel, not sequentially). Each agent
   prompt must be self-contained and tell it to:
   - Read `BRANDING.md`, `COMMUNICATION_STYLE.md`, the `## <network>` section of
     `STRUCTURES.md`, and `content/<NNN>-<slug>/RESEARCH.md`.
   - Use `.claude/skills/content-creator/templates/page-video.html` as the base for
     `youtube`/`instagram`/`tiktok`, or `templates/page-article.html` for `article`.
   - Follow the brand's tone and that network's structure exactly.
   - For video networks: fill in a full **Video Script** AND a **Caption & Hashtags**
     block (hashtags always; @mentions whenever the script names a specific person or
     brand).
   - For `article`: fill in the **Article** body only.
   - Write the result to `content/<NNN>-<slug>/pages/<network>.html`, replacing the
     `{{...}}` placeholders (including `{{TOPIC_TITLE}}` and `{{NETWORK}}`) — no
     placeholder text should remain in the output.

5. **Update the root index.** `index.html` is a single running index across ALL topics
   — never overwrite it wholesale.
   - If it doesn't exist yet, start from `.claude/skills/content-creator/templates/index.html`.
   - Read the current file, find the `<!-- CONTENT-CREATOR:TOPICS -->` marker comment.
   - If a `<section data-topic="<NNN>-<slug>">` for this topic already exists (revision
     case), replace just that section. Otherwise insert a new section right after the
     marker comment, preserving every other existing section untouched. Each section
     should show the topic title and a link to every generated `pages/<network>.html`.

6. **Report** to the user: the topic folder, the pages generated, and the updated index.

## Procedure — revising existing content

If the user refers to an existing topic/network (e.g. "change the hook on the YouTube
script for topic 001", "make the Instagram caption punchier"):

- Locate `content/<NNN>-<slug>/pages/<network>.html` directly (list `content/` if you
  need to resolve the number or slug) and edit it yourself with Read + Edit in the
  current conversation. Do not re-run research and do not touch other networks' files —
  each network's page is self-contained, so there's nothing else to reload.
- Only fall back to launching a fresh single-network subagent (step 4 above, for just
  that one network) if the request is big enough to need re-reading the full
  `RESEARCH.md` context (e.g. "redo this with a completely different angle").
- If the edit changes the page's title, update that topic's entry in `index.html` too.
