---
name: rephrase-simply
description: Rewrite text into ASD-STE100 Simplified Technical English (STE), honoring STE's length caps and any length target the user gives. Use when asked to rephrase, simplify, or rewrite text "simply", "plainly", "in STE", "in Simplified Technical English", "in plain English", or for a non-native / broad audience.
---

Rewrite the supplied text into ASD-STE100 Simplified Technical English (STE). Keep the meaning; change only the wording. Do not add facts, and do not drop facts, unless a length limit forces cuts — then cut the least important facts and say what you cut.

## Length rules (apply in this order)

1. **Honor an explicit target first.** If the user gives a length (e.g. "2–3 sentences", "one paragraph", "50 words"), that limit wins. Meet it exactly, even if it means merging or dropping detail.
2. **Apply STE sentence caps** within that target:
   - Instructions (procedures): **20 words** maximum per sentence.
   - Descriptions / statements: **25 words** maximum per sentence.
   - Keep to **one idea per sentence**. One instruction per sentence.
   - Keep paragraphs to **6 sentences** maximum (procedures) where practical.
3. If the two conflict (a tight target vs. many facts), meet the target and use STE-style short clauses inside it.

## STE writing rules

- **Active voice only.** Write "You must set the flag", not "The flag must be set".
- **Simple tenses.** Use the present, the past, or the future. Avoid perfect and continuous tenses (avoid "has been", "is being").
- **One word, one meaning.** Use each word with a single approved meaning. Do not use a word as both noun and verb (e.g. use "the oil" and "apply oil", not "to oil").
- **Approved vocabulary and short words.** Prefer the shortest common word. Replace "utilize" → "use", "prior to" → "before", "in order to" → "to", "assist" → "help", "commence" → "start", "terminate" → "stop".
- **No gerunds / -ing as nouns.** Rewrite "Cleaning the part is important" → "You must clean the part."
- **Keep articles.** Always write "the", "a", "an". Do not drop them.
- **No slang, idioms, jargon, or figures of speech.** Say the literal thing.
- **Consistent terminology.** Name the same thing the same way every time. Do not use synonyms for a technical term.
- **Spell out abbreviations** on first use, unless the abbreviation is standard for the audience.
- **Positive form for instructions** where possible. Prefer "Keep the cover closed" over "Do not open the cover" when both fit.
- **Warnings and cautions first.** If a step has a safety or failure risk, state the condition before the action.
- **Numbers and lists.** Use lists for parallel items and numbered steps for sequences. One action per step.

## Output

- Return the rewritten text only, unless the user asked for notes or a comparison.
- Match the requested format (PR description, comment, paragraph, bullets).
- If you cut facts to meet a length limit, add one short line after the text naming what you removed.
- If the source is ambiguous and STE forces a single meaning, pick the most likely reading and note the assumption in one line.
