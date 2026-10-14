---
name: content-creator
description: Research a topic and generate video scripts plus social captions (or an article) across the creator's configured networks, using the brand profile set up by brand-setup. Use when the user asks to create content, write a script, draft a post, or research a topic for their channel — e.g. "create content about X", "write me a YouTube script on X", "make a post about X". Also use for follow-up edits to already-generated content, e.g. "change the hook on the Instagram script for topic 001".
user-invocable: true
argument-hint: <topic>
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
  RESEARCH_SOURCE.md      # sources found (ID + name + title + URL + note + verbatim excerpts)
  RESEARCH.md             # synthesized findings on the topic, each fact tagged [S#]
  pages/
    <network>.html        # one per configured network, e.g. youtube.html, article.html
index.html                # root index of every topic, updated (never fully rebuilt) each run
```

`<NNN>` is a zero-padded sequence number (`001`, `002`, ...). `<topic-slug>` is the
topic kebab-cased (lowercase, spaces/punctuation → `-`).

Paths under `templates/` below are relative to this skill's own directory (the folder
holding this `SKILL.md`), wherever your agent installed it. Resolve them to absolute
paths before handing them to a subagent.

## Procedure — new topic

1. **Determine the topic and slug.** List `content/` to find the highest existing
   `<NNN>` prefix; the new one is that + 1 (use `001` if `content/` doesn't exist yet).
   Create `content/<NNN>-<slug>/` and `content/<NNN>-<slug>/pages/`.

2. **Determine target networks.** Read `MAIN_INFLUENCE_ZONE.md` and
   `SECONDARY_INFLUENCE_ZONE.md`. By default generate for all configured networks. If
   the user explicitly asked for only some networks in their request, honor that
   instead.
   - **If `instagram` is one of the target networks, also determine its format:**
     `reel`, `post` (regular feed image), or `carousel`. If the user's request already
     names one (e.g. "instagram carousel about X", "make a reel", "post for instagram"),
     use that. Otherwise ask a single-select question (with a structured question tool
     such as `AskUserQuestion` if your agent has one, otherwise a numbered list in
     chat) — one option each for Reel / Post / Carousel, with a one-line description of
     what each is — before moving on to research/generation.

3. **Research (one subagent).** Launch a single fresh subagent with web search, web
   fetch and file-write access (e.g. a general-purpose agent with WebSearch/WebFetch/
   Write in Claude Code) with a self-contained prompt along these lines. If your agent
   can't spawn subagents, do this step yourself with the same instructions:

   > Research the topic "`<topic>`" for a content creator. Find credible, current
   > sources (articles, official docs, reputable news, studies — avoid low-quality
   > SEO content). Write `content/<NNN>-<slug>/RESEARCH_SOURCE.md` with one entry per
   > source, each with a stable ID (`S1`, `S2`, ...), the source's name (outlet, author
   > or organization), the page title, the URL, a one-line note on what it's useful
   > for, and one or more **verbatim excerpts** — the exact passages, copied word for
   > word in their original language and wrapped in quotes, that back each fact you
   > take from that source. Never paraphrase inside an excerpt. Then write
   > `content/<NNN>-<slug>/RESEARCH.md`: a synthesized write-up of the topic covering
   > key facts, useful angles/hooks for content, and anything surprising or
   > counter-intuitive worth highlighting. Tag every fact, number, quote or notable
   > claim with the ID of the source it came from (e.g. `[S3]`). Do not write any HTML
   > or scripts — only these two Markdown files.

   Wait for it to finish before continuing. This keeps raw search results out of your
   own context — you only need to know it succeeded.

4. **Generate one page per network (parallel subagents).** Launch one fresh subagent
   per target network, all in parallel, not sequentially (in Claude Code: all in a
   single message). If your agent can't spawn subagents, generate the pages yourself
   one network at a time, finishing and writing each before starting the next. Each
   subagent prompt must be self-contained and tell it to:
   - Read `BRANDING.md`, `COMMUNICATION_STYLE.md`, the `## <network>` section of
     `STRUCTURES.md`, `content/<NNN>-<slug>/RESEARCH.md`, and
     `content/<NNN>-<slug>/RESEARCH_SOURCE.md`.
   - Use as the base template:
     - `templates/page-video.html` for `youtube`,
       `tiktok`, and `instagram` when its format is `reel`.
     - `templates/page-instagram-post.html` when
       `instagram`'s format is `post`.
     - `templates/page-instagram-carousel.html` when
       `instagram`'s format is `carousel`.
     - `templates/page-article.html` for `article`.
   - Follow the brand's tone and that network's structure exactly.
   - **For `youtube`, `tiktok`, and `instagram` reels — the Video Script must be a
     shot-by-shot recording script, not narration.** Keep the macro beats from
     `STRUCTURES.md` (Hook/Intro/Development/Conclusion, etc.) as section headers, but
     within each beat interleave the spoken lines with bracketed production cues for
     every camera take/framing, cut, b-roll/image insert, on-screen text, and
     sound/music moment — placed exactly where it happens, not summarized afterward.
     Never leave a whole beat as an undifferentiated paragraph of speech with no visual
     or sound direction. For example:
     ```
     [take de câmera, olhando direto pra lente]
     Ninguém tá cobrando de fora — você que não se dá por satisfeito.

     [corta pra imagem de arquivo do pôr do sol, zoom lento]
     [toca uma trilha suave de fundo]

     [volta pro rosto]
     Christopher Ash chama isso de esquecer que você é pó.
     ```
     Then add the **Caption & Hashtags** block (hashtags always; @mentions whenever the
     script names a specific person or brand).
   - **For `instagram` posts:** fill in a **Visual Concept** section describing the
     single image/graphic to produce (composition, any on-image text, mood) instead of
     a video script, then the **Caption & Hashtags** block.
   - **For `instagram` carousels:** fill in a **Carousel Slides** section with one
     numbered slide per card (visual concept + on-slide text for each — default to
     roughly 5–8 slides unless the topic clearly needs more or fewer), then the
     **Caption & Hashtags** block.
   - **For `article`:** fill in the **Article** body only.
   - **Every network and format — show the source of each reference inline.** Right
     after each piece of content that uses something from the research (a spoken line,
     a slide, the visual concept, an article paragraph, the caption), insert a source
     block using the `<aside class="source-ref">` snippet documented in the template's
     comment. It holds the verbatim excerpt from `RESEARCH_SOURCE.md` (follow the
     `[S#]` tag in `RESEARCH.md` to find it), the source's name and title, and the link.
     Write the block's label in the content's language (e.g. "Fonte · só para
     referência, não entra no conteúdo"). The block is for the creator to check where
     something came from. It is not part of what gets recorded or published, and the
     template styles it so it reads that way. A reference used twice gets a block both
     times. Content that doesn't come from the research (the creator's own opinion,
     production cues) gets no block.
   - **For `instagram` specifically** (any format): add an HTML comment
     `<!-- instagram-format: reel|post|carousel -->` right after the opening `<body>`
     tag, recording the format used, so future revisions know it without re-asking.
   - Write the result to `content/<NNN>-<slug>/pages/<network>.html`, replacing the
     `{{...}}` placeholders (including `{{TOPIC_TITLE}}` and `{{NETWORK}}`) — no
     placeholder text should remain in the output.

5. **Update the root index.** `index.html` is a single running index across ALL topics
   — never overwrite it wholesale.
   - If it doesn't exist yet, start from `templates/index.html`.
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
  need to resolve the number or slug) and edit it yourself in the current
  conversation. Do not re-run research and do not touch other networks' files —
  each network's page is self-contained, so there's nothing else to reload.
- For `instagram`, first read the `<!-- instagram-format: ... -->` comment near the top
  of the file to know its current format. If the request implies switching format
  entirely (e.g. "turn this into a carousel", "make it a regular post instead"), don't
  edit in place — regenerate it via step 4 above using the new format's template.
- Only fall back to launching a fresh single-network subagent (step 4 above, for just
  that one network) if the request is big enough to need re-reading the full
  `RESEARCH.md` context (e.g. "redo this with a completely different angle").
- Keep each `<aside class="source-ref">` block next to the content it backs. If an
  edit brings in a new fact from the research, add its source block (from
  `RESEARCH_SOURCE.md`). If an edit removes content that had a source block, remove
  that block too.
- If the edit changes the page's title, update that topic's entry in `index.html` too.
