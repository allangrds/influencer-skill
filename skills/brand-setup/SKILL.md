---
name: brand-setup
description: Set up or update the creator's brand profile — brand info, communication tone, citation style, main and secondary content networks, inspirations, and per-network content structure. Use when the user wants to configure, set up, or edit their creator/branding profile before generating content, or says things like "configure my brand", "set up my content profile", "update my tone/style", "change my citation style", "change my networks", "add an inspiration".
user-invocable: true
---

# Brand Setup

Interviews the creator and writes their branding configuration to a fixed set of
Markdown files at the **project root** (not inside the agent's config or skills
directory, e.g. `.claude/` or `.agents/`). These files are the
single source of truth that the `content-creator` skill reads from later — keep their
names and locations exactly as below.

## Output files (project root)

| File | Holds |
|---|---|
| `BRANDING.md` | Name, what they do, audience, free-form brand notes |
| `COMMUNICATION_STYLE.md` | Tone of voice and citation style |
| `MAIN_INFLUENCE_ZONE.md` | The single primary network |
| `SECONDARY_INFLUENCE_ZONE.md` | Zero or more secondary networks |
| `INSPIRATIONS.md` | Creators/brands/content that inspire this creator |
| `STRUCTURES.md` | Desired content structure per configured network |

Valid networks (always use these exact lowercase slugs, in English, since they double as
filenames later): `article`, `youtube`, `instagram`, `tiktok`.

Templates for all six files live in `templates/` next to this SKILL.md. Copy a template
to the project root and fill it in — don't invent a different structure or heading set.

## Procedure

1. **Check what already exists.** For each of the six output files, check if it exists
   at the project root and already has real content (not just the untouched template).
   - If ALL six already have content, tell the user their brand is already configured,
     summarize the current settings briefly, and ask whether they want to update
     anything specific (e.g. "just add an inspiration", "change my tone") or redo a
     section from scratch. Jump straight to the relevant step below — don't re-run the
     full interview.
   - If some are missing/empty, only interview for those; skip the ones already filled.
   - If `COMMUNICATION_STYLE.md` has content but no filled-in `## Citation style`
     section (it was created before that section existed), ask only the citation
     question from step 3 and add the section.

2. **Branding** (`BRANDING.md`) — ask, in conversation, for: the brand/creator name,
   what they do, their audience, and anything else they want to share. Copy
   `templates/BRANDING.md` to the project root and fill it in with their answers,
   keeping their own wording where possible.

3. **Communication style** (`COMMUNICATION_STYLE.md`) — ask them to describe (or give
   examples of) how they talk to their audience: tone, humor, pacing, words/phrases to
   avoid. Then ask which standard to use when citing sources, as a single-select
   question (structured question tool if available, otherwise a numbered list): ABNT
   (NBR 6023 + NBR 10520:2023), APA 7, Chicago author-date, or another one they name.
   Mark as recommended ABNT if they create content in Portuguese, APA 7 otherwise. Fill
   in `templates/COMMUNICATION_STYLE.md` at the project root, writing the chosen
   standard and its edition under `## Citation style`.

4. **Main network** (`MAIN_INFLUENCE_ZONE.md`) — ask a single-select question (with a
   structured question tool such as `AskUserQuestion` if your agent has one, otherwise
   a numbered list in chat) with the four network options (`article`, `youtube`,
   `instagram`, `tiktok`) to pick the ONE primary network. Write the chosen slug into
   `MAIN_INFLUENCE_ZONE.md`.

5. **Secondary networks** (`SECONDARY_INFLUENCE_ZONE.md`) — ask a multi-select question
   the same way (allow selecting none) with the remaining three networks. Write the
   chosen slugs (or an empty list) into `SECONDARY_INFLUENCE_ZONE.md`.

6. **Inspirations** (`INSPIRATIONS.md`) — ask for names and/or links of creators, brands,
   or specific content that inspire them. One bullet per inspiration.

7. **Structures** (`STRUCTURES.md`) — for the main network and every secondary network
   chosen in steps 4–5, ask what structure they want for that type of content. Build a
   single `STRUCTURES.md` with one `## <network>` section per configured network
   (delete sections for networks not in use — see `templates/STRUCTURES.md`). Enforce
   this rule regardless of what the user says first:
   - `youtube` / `instagram` / `tiktok` → the section MUST cover two parts: the **video
     script** structure, and the **caption/support-text** structure (must include
     hashtags, and @mentions whenever the content references a specific person or
     brand).
   - `article` → the section covers only the **article body** structure — no video
     script, no hashtags.

8. **Confirm.** List the files that were created or updated so the user can see the
   final state. Do not proceed to generate any content in this skill — that's the job
   of the `content-creator` skill.
