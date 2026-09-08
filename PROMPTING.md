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

### Artist tags

**Prefix with `@` or the effect is weak.** The card is emphatic. `@nnn yryr`, not `nnn yryr`.

On 2.9B this is the main fix for flatness. The new blocks push toward glossy saturated commercial illustration with thin shadows. An artist tag pulls it back toward a specific look. In a published CFG sweep, raising CFG from 4 to 6 did **not** restore the shading, while a single artist tag changed line weight, ornament detail and composition outright.

### Character and series

**Always pair a character tag with its series tag.** Names collide across franchises and the model confuses them without the copyright tag. Both cards say this.

For multiple characters, name each **and describe their appearance**. Anima reliably places attributes in the positions you specify and just as reliably gets the identities wrong if you only give names.

---

## 4. Natural language mode

Pure prose, pure tags, or any mix, in any order.

- Standard English capitalization for character and series names. `Fern from Sousou no Frieren`.
- At least two sentences. Very short prose gives unexpected results.
- Quality and artist tags can lead a prose prompt: `masterpiece, best quality, @big chungus. An anime girl with medium-length blonde hair is...`
- Name a character, then describe their appearance. Essential for multi-character scenes.

```
Digital artwork of Fern from Sousou no Frieren, with long purple hair and
purple eyes, wearing a black coat over a white dress with puffy sleeves...
```

### Dataset tags, an obscure feature almost nobody uses

Anima-Base was additionally trained on LAION-POP (ye-pop) and DeviantArt, both photo-filtered. Those captions carried a dataset tag on its own line, optionally followed by a title line. You can invoke that mode:

```
deviantart
Flame
Digital painting of a fiery dragon with glowing yellow eyes, black horns and a
long sinuous tail, perched on a glowing molten rock formation.
```

`deviantart` for painterly digital-art conventions, `ye-pop` for a broader illustration-stock look. A real style lever for non-anime illustration.

---

## 5. Prompt weighting

Works, but needs **much heavier weights than SDXL**. The card's own example is `(chibi:2)`. If you are used to `(thing:1.2)`, that is close to a no-op here. Start at 1.5.

---

## 6. Negative prompts

Official Base recommendation:

```
worst quality, low quality, score_1, score_2, score_3, artist name, blurry,
jpeg artifacts, chromatic aberration
```

For 2.9B, drop the score tags and add safety tags:

```
worst quality, low quality, lowres, jpeg artifacts, chromatic aberration,
signature, artist name, watermark, username, text,
bad anatomy, extra digits, missing finger,
nsfw, sensitive, explicit
```

`bad anatomy`, `extra digits` and `missing finger` **are** real Danbooru tags, so they belong. The rest of the usual copy-paste block does not.

A negative prompt is not a wishlist. Every token competes for conditioning. Ten chosen tags beat forty scraped off a Civitai post.

---

## 7. What does not work

**Negation inside the positive prompt.** "no additional elements", "no change to the framing", "without a background" suppress nothing. Diffusion conditioning has no logical NOT. Worse, the tokens for the thing you are excluding are now in your positive conditioning, so you can summon exactly what you were trying to avoid. Exclusions go in the negative, as plain tags.

**Invented tags.** If it is not on Danbooru or Gelbooru, the model never associated it with anything. Check before you use it.

**SDXL-scale weights.** See section 5.

**Realism.** Explicitly out of scope. The card says the model will not work well at realism, by design. There is a community 2.9B finetune (`addansee/Anima-2.9B-RealAddn`, 2100 real photos) if you need it.

**Long text rendering.** Single words sometimes, short phrases occasionally, sentences no.

---

