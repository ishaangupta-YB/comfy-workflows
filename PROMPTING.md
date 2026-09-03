# Anima / Anima-2.9B prompting reference

Sourced from the official Anima-Base and Anima-2.9B model cards plus community testing. Where 2.9B differs from Base, it is called out. Where something is folklore rather than fact, that is called out too.

---

## 1. Why this model prompts differently from SDXL

Anima does not use CLIP. The text path is:

```
your prompt
  -> Qwen3-0.6B          a base language model, not an embedding model
  -> llm_adapter         a small transformer that reshapes the LLM output
  -> cross-attention in each of the DiT's 28 (Base) or 40 (2.9B) blocks
```

Three consequences that invalidate most SDXL habits.

**No 77-token limit.** No chunking, no BREAK, no per-chunk weight weirdness. Long prompts are native, and on 2.9B they are required. Short prompts produce bland flat backgrounds.

**Real language parsing.** A base LLM handles grammar, clauses and word order. "The girl on the left has red hair, the girl on the right has blue hair" largely works. SDXL could not do that.

**But the vocabulary is still Danbooru.** Invented tags are not neutral, they are out-of-distribution noise. `bad hands` is not a Danbooru tag. Neither is `fewer digits`, `deformed`, `mutated`, `poorly drawn`. That is SD1.5-era folklore that got copy-pasted forward for four years. Here it contributes nothing and eats conditioning budget.

The `llm_adapter` is the most fragile component in the stack. Official finetuning advice is to never train it, because it holds a surprising amount of knowledge and degrades easily. Remember that for the finetuning guide.

---

## 2. Tag order

A hard structure, not a suggestion. The captions were written this way.

```
[quality] [meta] [year/period] [safety]  [1girl/1boy/1other]  [character]  [series]  [@artist]  [general]
```

Order within a section does not matter. Order across sections does.

Official worked example:

```
year 2025, newest, normal quality, score_5, highres, safe,
1girl,
oomuro sakurako,
yuru yuri,
@nnn yryr,
smile, brown hair, hat, solo, fur-trimmed gloves, open mouth, long hair,
gift box, fang, skirt, red gloves, blunt bangs, one eye closed, shirt,
santa costume, red hat, skin fang, white background, holding bag,
fur trim, simple background, brown skirt, bag, looking at viewer,
santa hat, ;d, red shirt, box, gift, holding, red capelet, capelet
```

Lowercase. Spaces, not underscores. Score tags are the one exception and keep their underscore.

Where Danbooru and Gelbooru disagree on a tag name, use the Gelbooru version.

---

## 3. Each section

### Quality tags

Two independent systems. Use either, both, or neither.

| System | Values |
|---|---|
| Human score | `masterpiece`, `best quality`, `good quality`, `normal quality`, `low quality`, `worst quality` |
| PonyV7 aesthetic model | `score_9` down to `score_1` |

**The 2.9B difference that matters:** its training captions contained **no score tags at all**. The card says you can still use them, which is true only because the frozen original 28 blocks still carry them. The 12 new blocks have never seen one. Expect weaker and less predictable behavior than on Base. I would drop score tags from 2.9B prompts entirely and use the human-score words.

On Anima-Aesthetic the card recommends dropping `score_*` from both positive and negative, because the model is already aesthetic-tuned and pushing harder sends it into slop.

### Time period tags

Underrated, and the strongest single lever on art style.

```
year 2025, year 2024, year 2023, ...
newest, recent, mid, early, old
```

`old` gets 2000s anime aesthetics. `newest` gets current Pixiv. If your output looks dated and you did not ask for that, you left the period tag off and the model averaged twenty years of art.

### Meta tags

```
highres, absurdres, anime screenshot, anime coloring, official art, jpeg artifacts
```

`anime screenshot` plus `anime coloring` is the pair that produces a TV-anime look instead of an illustration look. Community LoRA trainers deliberately tag screencaps this way because Anima holds the style well inside those two tags. Directly relevant if you are training on anime frames.

### Safety tags

```
safe, sensitive, nsfw, explicit
```

`safe` in positive, the other three in negative. The card lists unwanted content as a known limitation and names safety tags as the mitigation. This matters most with short prompts, because a short prompt leaves the model free to fill in.

