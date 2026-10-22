# Qualifier prompt

This is the scoring prompt used by the **Qualify Company (LLM)** node. It has three parts:

1. **ICP block** – who you sell to. In the workflow this text is *not* hard-coded in the
   prompt: it is read from the `icp` field of the **Config** node, so you change your ICP in one
   place. `[[ICP]]` below marks where it is injected.
2. **Rubric** – how a 1–10 score is assigned. Edit it to match how your team grades fit.
3. **Output rules** – constraints on each field. The exact JSON shape is enforced separately by
   the **Structured Output Parser** node (see the schema at the bottom of this file).

> Keep curly braces out of the prompt. The LLM node passes the prompt through a LangChain
> prompt template, where `{` and `}` are template variables. The workflow also strips braces from
> the scraped website text for the same reason.

## Example ICP (fictional)

```text
B2B SaaS companies in the US with 20-500 employees that sell to marketing teams.
```

Good ICP text is short and testable. Name the business model, the geography, the size band and
the buyer. Add explicit exclusions if you have them, for example
"Exclude agencies, consultancies and marketplaces."

## System prompt

The text between the markers is exactly what the workflow's system message contains. The only
difference is that `[[ICP]]` is replaced by the expression `{{ $('Config').first().json.icp }}`.
If you edit the prompt here, paste the same change into the node's system message (and the reverse).

<!-- SYSTEM_PROMPT_START -->
You are a B2B sales research analyst. You qualify companies against an Ideal Customer Profile (ICP) using only the website content you are given.

IDEAL CUSTOMER PROFILE:
[[ICP]]

SCORING RUBRIC (score is an integer from 1 to 10):
- 9-10: Clear match on every ICP dimension (business model, geography, size, buyer). Strong evidence on the site.
- 7-8: Matches the core of the ICP; one dimension is uncertain or slightly outside the range.
- 5-6: Partial match. Plausible fit, but important dimensions are missing or ambiguous.
- 3-4: Weak match. Adjacent business, wrong buyer, or clearly outside the size or geography range.
- 1-2: No match, or the content does not describe a business at all.

RULES:
- Base every judgement on the provided content. Do not invent facts, customers, funding or headcount.
- When evidence is missing, say so in the reasons and lower the score; never guess upward.
- industry: a short label, 1 to 4 words, for example "Marketing analytics SaaS".
- company_size_guess: one of 1-10, 11-50, 51-200, 201-500, 501-1000, 1000+, unknown. Use unknown when the site gives no signal.
- reasons: 2 to 5 short bullet-style sentences explaining the score, each under 25 words.
- pitch: one paragraph, at most 80 words, addressed to the company. Reference something specific from their site and connect it to a concrete outcome. No greetings, no sign-off, no placeholders such as [Name].
- Write in English.
<!-- SYSTEM_PROMPT_END -->

## User message

The user message is built per company inside the workflow:

```text
Qualify this company against the Ideal Customer Profile.

Company name: <company_name from the sheet>
Website: <normalized URL>
Content truncated to fit the budget: <true|false>

Website content (markdown, may be partial):
"""
<cleaned markdown from Jina Reader>
"""
```

## Output schema (Structured Output Parser)

```json
{
  "type": "object",
  "properties": {
    "score": { "type": "integer", "minimum": 1, "maximum": 10 },
    "industry": { "type": "string" },
    "company_size_guess": {
      "type": "string",
      "enum": ["1-10", "11-50", "51-200", "201-500", "501-1000", "1000+", "unknown"]
    },
    "reasons": { "type": "array", "items": { "type": "string" }, "minItems": 1, "maxItems": 5 },
    "pitch": { "type": "string" }
  },
  "required": ["score", "industry", "company_size_guess", "reasons", "pitch"],
  "additionalProperties": false
}
```

The parser rejects output that does not match the schema. Even when the output matches,
**Validate Result** still clamps the score to 1–10, coerces `reasons` to a list of at most five
strings, maps any unexpected size value to `unknown`, and caps the pitch length. The LLM is
treated as untrusted input.

## Tuning tips

- **Too many 7+ scores?** Make the 7–8 band stricter ("matches *every* dimension except one,
  and that one is unknown rather than wrong").
- **Generic pitches?** Require the pitch to quote or paraphrase one concrete product feature
  from the site.
- **Different buyer per product line?** Add one line per buyer persona to the ICP and ask the
  model to name the matched persona as the first reason.
