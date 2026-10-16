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
  RESEARCH_SOURCE.md      # sources found (ID + name + title + URL + note + verbatim excerpts + full reference + citation key)
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
   - **Determine the citation style:** read the `## Citation style` section of
     `COMMUNICATION_STYLE.md` (e.g. ABNT, APA 7). If it's missing or empty, use ABNT
     (NBR 6023 + NBR 10520:2023) when the brand's content is in Portuguese and APA 7
     otherwise, and mention in the final report that it can be set with `brand-setup`.
     Pass the style to every subagent below.

3. **Research (one subagent).** Launch a single fresh subagent with web search, web
   fetch and file-write access (e.g. a general-purpose agent with WebSearch/WebFetch/
   Write in Claude Code) with a self-contained prompt along these lines. If your agent
   can't spawn subagents, do this step yourself with the same instructions:

   > Research the topic "`<topic>`" for a content creator. Find credible, current
   > sources (articles, official docs, reputable news, studies — avoid low-quality
   > SEO content); prefer the primary source (the study, the law, the author's own
   > book) over articles about it. Write `content/<NNN>-<slug>/RESEARCH_SOURCE.md` with
   > one entry per source, each with a stable ID (`S1`, `S2`, ...), the source's name
   > (outlet, author or organization), the page title, the URL, a one-line note on what
   > it's useful for, and one or more **verbatim excerpts** — the exact passages, copied
   > word for word in their original language and wrapped in quotes, that back each
   > fact you take from that source. Never paraphrase inside an excerpt. Each entry also
   > gets a `Reference:` line — the complete reference in `<citation style>`, current
   > edition, for that kind of source (book, journal article, review, news story, web
   > page, survey/report, law/official document, interview, Bible), every field taken
   > from the source itself (author, title, outlet, city, date, volume/issue/pages, DOI,
   > URL and access date when the style asks for them) — and a `Citation:` line with the
   > in-text citation, e.g. `(Fontes, 2020, p. 5)` in ABNT, which since NBR 10520:2023
   > writes names in upper and lower case. Never invent a field: leave out what can't be
   > confirmed and say what's missing. Watch the usual traps: no personal author →
   > entry by institution or by title; an interview → under the interviewee; same
   > author and year → `2016a`, `2016b`. If an author's quote or idea was found only in
   > a secondary source (a review, a summary), add a `Secondary:` line naming the
   > original work, so the content cites it as the style requires (`apud` in ABNT,
   > "as cited in" in APA). Then write
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
   - **Every network and format — a source box for the creator.** Right after each
     piece of content that uses something from the research (a spoken line, an
     on-screen text, a slide, the visual concept, an article paragraph or quote, the
     caption), insert the source box documented in the template's comment, and put the
     template's legend once at the top of each section that has boxes. The box is for
     the creator to fact-check; it is never recorded or published, and the template
     styles it so it reads that way. Write its label, legend and field names in the
     content's language — e.g. Portuguese: "📎 Fonte — não faz parte da fala" (video
     scripts) / "📎 Fonte — não faz parte do texto" (everything else), "O que diz",
     "⚠️ Atenção", "Conferir"; English: "📎 Source — not part of the script" / "📎 Source
     — not part of the text", "What it says", "⚠️ Note", "Check". Fields:
     - **What it says:** a complete paraphrase of what the source says — who, when,
       where, what, context — with the verbatim excerpt(s) from `RESEARCH_SOURCE.md`
       (follow the `[S#]` tag in `RESEARCH.md`) in quotes, with the page when there
       is one.
     - **⚠️ Note:** only when needed — what in the content is the creator's own
       paraphrase or illustration rather than the source, what couldn't be
       confirmed, what comes second-hand, wording changed to match the source.
     - **Check:** the full reference of every source the box relies on, copied from
       its `Reference:` line, with the URL as a link inside the reference. This holds
       for every kind of source, not only books: articles, reviews, studies, surveys,
       news, official pages, the Bible. Separate several with ` · `.

     A reference used twice gets a box both times. Content that doesn't come from the
     research (the creator's own opinion, production cues) gets no box. A claim with no
     source at all is removed, not flagged.
   - **Every network and format — a formal citation in what gets published**, in the
     brand's citation style, so anyone can look the source up later:
     - `article`: an in-text citation after every fact, number, quote or idea taken
       from a source, copied from its `Citation:` line — in ABNT `(Han, 2024, p. 17)`,
       `Han (2024, p. 17)`, `(Organização Mundial da Saúde, 2019)`, a second-hand one
       `(Ash, 2016 apud Challies, 2016)`; in APA `(Han, 2015, p. 17)`, `(Ash, 2016, as
       cited in Challies, 2016)`. Sacred or classical texts by book, chapter:verse and
       version, e.g. `— 1 Reis 19:4 (ARA)`. End with a references section (heading in
       the content's language, e.g. "Referências", "References"): the full reference
       of every cited source and only those, ordered and formatted as the style says,
       including the edition of any sacred text quoted.
     - Video scripts (`youtube`, `tiktok`, instagram reels): the spoken line stays
       natural ("segundo a OMS…", "according to the WHO…"), but whenever a study,
       figure, book or quote comes in, add a production cue for an on-screen citation
       in the content's language, e.g. `[citação na tela, no rodapé: Han (2024, p. 19)
       — Sociedade do cansaço]` / `[on-screen citation, lower third: Han (2015, p. 19)
       — The Burnout Society]`. In the `youtube` caption, add the references list: the
       full reference of every source the script cites, plus a line with any Bible
       verses cited.
     - `tiktok` and `instagram` (any format): the citation on screen, on the slide or
       on the image, plus a caption line pointing to the full list — the YouTube
       video's description if this topic has a `youtube` page, otherwise the article
       if it has an `article` page, otherwise a short references list in the caption
       itself.
   - **Before writing the page, check it:** every box has a "Check" field with a full
     reference and its link; in an `article`, every in-text citation has an entry in
     the references section and every entry is cited; no claim is left without a
     source.
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
- Keep each source box next to the content it backs. If an edit brings in a new fact
  from the research, add its source box (from `RESEARCH_SOURCE.md`) and its formal
  citation, and add the source to the article's references section or the YouTube
  description if it isn't there yet. If an edit removes content that had a source box,
  remove that box too, and drop the source from those lists when nothing else cites
  it.
- If the edit changes the page's title, update that topic's entry in `index.html` too.
