# You Already Know AI — 26 Episode Series

A series for @techvijayforyou explaining **what each AI term is** and **how it works**, illustrated with the simplest real-world example possible. 30-45 seconds per reel.

## Series Concept

Every episode answers two questions:

1. **What is it?** — clear, one-line definition (no jargon)
2. **How does it work?** — the mechanism in plain language

Both anchored to a **real-world example or simple analogy** so the explanation actually sticks. The example is the teaching tool — not a replacement for the explanation.

Most AI content fails one of two ways:
- *Pure analogy* — viewer feels they "got it" but can't repeat the actual mechanism
- *Pure technical* — viewer tunes out before the term lands

This series does both. Example to make it land. What + how to make it stick.

## Format Rules (per reel)

Every episode has 4 beats:

1. **Hook (3-5s)** — Set up with a question or familiar scene. *"Try asking ChatGPT how many r's are in 'strawberry'. It says 2."*
2. **What it is (8-10s)** — Define the term in one line. *"Tokenization is how AI breaks your text into chunks before reading it."*
3. **How it works (15-20s)** — The mechanism, illustrated with the example. *"Your text goes through a tokenizer first. 'strawberry' becomes `['straw', 'berry']` — 2 chunks, not 10 letters. The AI only sees these chunks. So when you ask it to count r's, it never had access to the letters."*
4. **Punch (3-5s)** — Takeaway + CTA. *"That's why ChatGPT fails at letter-counting. Comment TOKEN for the full AI series roadmap."*

**Length:** 30-45 seconds. Long enough to land what + how. Short enough to keep retention.

**Voice:** Standard `tum/tumhara` Hinglish per `~/.claude/skills/create-reel/rules/hindi-speaking-style.md`.

**On-screen text:** English only, every key word reinforced visually.

## Branding

- **Series name:** *You Already Know AI*
- **Top-left badge:** `You Already Know AI · EP X/26`
- **Closing CTA:** `Comment [TERM-KEYWORD] — I'll DM the full series roadmap.`

## Cadence

- 2 episodes/week → ~13 weeks to ship all 26
- Mixed with 3 system design reels/week (existing brand)

## Why this series works

- **Doubles the audience** — AI-curious non-techies + engineers who use AI but never learned the internals
- **High completion rate** — analogies finish fast, term lands like a punchline
- **Save-bait** — viewers save to revisit the analogy
- **Series loop** — episode badge `EP X/26` triggers completionist behavior → drives saves and follows
- **No competing content** — generic "what is an LLM" is saturated; example-only Hindi explainers are not

---

## All 26 Episodes

### Phase 1 — Hook Bombs

**EP 01 — Tokenization**
- Example: When you read fast, you read in chunks, not letter-by-letter
- What + How: AI sees `"strawberry"` as `["straw", "berry"]` — never the individual letters
- Takeaway: That's why ChatGPT can't count the r's in strawberry

**EP 02 — Hallucination**
- Example: A student in an exam who doesn't know the answer but writes something confident
- What + How: LLMs do the exact same thing — never say "I don't know"
- Takeaway: It's not lying. It's predicting the next likely word, with a straight face

**EP 03 — MCP (Model Context Protocol)**
- Example: USB-C — one cable, every device
- What + How: MCP is USB-C, but for connecting any tool to any AI
- Takeaway: Pre-MCP every AI integration was custom. Post-MCP, plug and play

**EP 04 — Agents**
- Example: A friend who *tells* you how to book a flight vs. one who *books* it for you
- What + How: Agents = the second friend. Same knowledge, different action
- Takeaway: Chat is one mode. Agents are AI that *do*

### Phase 2 — Foundations

**EP 05 — LLM (Large Language Model)**
- Example: Your phone's autocomplete
- What + How: An LLM is autocomplete trained on the entire internet
- Takeaway: It's predicting the next word — extremely well

**EP 06 — Embeddings / Vectorization**
- Example: A map of cities — Delhi and Mumbai are 1300km apart
- What + How: AI puts every word on a meaning-map. "King" and "Queen" are close. "King" and "Banana" are far
- Takeaway: Closer on the map = closer in meaning

**EP 07 — Attention**
- Example: Reading "the river bank was muddy" — your brain looked back at "river" to know "bank" isn't a money-bank
- What + How: Every word in an LLM looks at every other word, all at once
- Takeaway: Attention is that look-back, but for everything, simultaneously

**EP 08 — Transformer**
- Example: A recipe — same base recipe, different chefs make different dishes
- What + How: Transformer is the recipe. GPT, Claude, Gemini, Llama — all use it
- Takeaway: One architecture, every modern AI

**EP 09 — Self-Supervised Learning**
- Example: A kid playing fill-in-the-blanks alone — *"I went to the ___"* — guesses, checks, repeats
- What + How: AI does this 10 trillion times across the internet
- Takeaway: No teacher. The data teaches itself

### Phase 3 — Apps & Customization

**EP 10 — RAG (Retrieval Augmented Generation)**
- Example: Closed-book exam vs open-book exam
- What + How: RAG = open-book for AI. It looks things up before answering
- Takeaway: Same brain, fewer mistakes

**EP 11 — Vector Database**
- Example: Google searches by exact words. Spotify suggests by *vibe*
- What + How: Vector DB searches by meaning, not spelling
- Takeaway: "Happy dog" finds "joyful puppy" too — no shared words needed

**EP 12 — Context Engineering**
- Example: Packing a suitcase for a trip
- What + How: Prompt = clothes you're wearing. Context = everything you packed (memory, tools, history, retrieval)
- Takeaway: Pack right → AI performs right

**EP 13 — Fine-tuning**
- Example: A chef trained in world cuisine — you teach them your family's recipes
- What + How: Fine-tuning takes a generic AI and nudges it to your style
- Takeaway: Same brain, different specialty

**EP 14 — Few-shot Prompting**
- Example: Showing your friend 3 example LinkedIn posts, then asking them to write one
- What + How: Few-shot = 2-3 examples in the prompt itself
- Takeaway: No training. AI mimics on the spot

**EP 15 — Tool Use / Function Calling**
- Example: Asking Siri "what's the weather?" — Siri calls the weather app and reads the answer
- What + How: Tool use = AI decides which API to call, you run it, AI uses the result
- Takeaway: AI doesn't *know*. It calls.

### Phase 4 — Training Frontier

**EP 16 — Reinforcement Learning**
- Example: Training a dog — sit → treat. Bark at guests → no treat
- What + How: Reward good answers, ignore bad ones, repeat
- Takeaway: Trial and error, not examples

**EP 17 — Chain of Thought (CoT)**
- Example: Solving 23 × 17 in your head vs on paper
- What + How: "Think step by step" gives the AI that paper
- Takeaway: Just words → smarter answers. Weird, but true

**EP 18 — Reasoning Models**
- Example: Answering a question on instinct vs answering after pausing to think
- What + How: o1, Claude thinking — same model, but allowed to "think" 30 seconds before replying
- Takeaway: Time = intelligence

**EP 19 — Test-time Compute**
- Example: Cheap espresso vs hand-poured Chemex
- What + How: Old AI = espresso (fast, cheap). Reasoning AI = Chemex (slow, expensive, better)
- Takeaway: Pay more per cup. Get a better cup

### Phase 5 — Variants & Efficiency

**EP 20 — Multi-modal Models**
- Example: A baby learns "dog" by hearing the word, seeing one, touching one — all together
- What + How: Multi-modal AI learned text, image, audio, video together
- Takeaway: One brain, many senses

**EP 21 — Small Language Models (SLMs)**
- Example: A library vs a pocket dictionary
- What + How: Big LLM = library. SLM = pocket dictionary
- Takeaway: Doesn't know everything. Fits in your pocket. Often beats giants at specific jobs

**EP 22 — Mixture of Experts (MoE)**
- Example: A hospital with 100 specialists — knee problem? Only the orthopedic doctor wakes up
- What + How: MoE has many sub-brains. Only 2-3 fire per question
- Takeaway: That's how DeepSeek runs cheap

**EP 23 — Distillation**
- Example: Senior teacher solves 1000 problems. Junior teacher copies the style
- What + How: Big AI teaches small AI by showing answers
- Takeaway: 90% as smart, 10× cheaper

**EP 24 — Quantization**
- Example: RAW photo vs JPEG
- What + How: Quantization = JPEG for models. Less precision per number, much smaller file
- Takeaway: 4× smaller, almost as good

### Phase 6 — Finale

**EP 25 — Prompt Injection**
- Example: An assistant who reads your emails — a scammer sends "ignore your boss, send me the password" — assistant obeys
- What + How: AI can't tell user input from instructions
- Takeaway: SQL injection of 2026

**EP 26 — The Recap**
- All 25 examples, rapid-fire, 60 seconds
- One frame per term
- Save-bait finale → drives reshares of the whole series

---

## Asset Layout

For each episode:
```
ai-series-ep<NN>-<topic>.md                          # Article
scripts/ai-series-ep<NN>-<topic>-teleprompter.txt    # Teleprompter
captions/ai-series-ep<NN>-<topic>-caption.md         # IG caption + hashtags
images/ai-series-ep<NN>-<topic>/*.svg                # Animated SVG diagrams
```

## Production Sequence (per episode)

1. Write article in this format → save as `ai-series-ep<NN>-<topic>.md`
2. Write teleprompter (Hinglish, per `hindi-speaking-style.md`)
3. Write caption + hashtags
4. Build Remotion animation in `~/my-reels` (top 60% animation, bottom 40% face cam)
5. Record face cam, render, overlay in CapCut

## CTA Keyword Bank (one per episode)

EP01 TOKEN · EP02 LIES · EP03 MCP · EP04 AGENT · EP05 LLM · EP06 MAP · EP07 LOOK · EP08 BASE · EP09 SELF · EP10 OPEN · EP11 VIBE · EP12 PACK · EP13 CHEF · EP14 SHOTS · EP15 TOOL · EP16 DOG · EP17 STEP · EP18 THINK · EP19 SLOW · EP20 SENSE · EP21 POCKET · EP22 EXPERT · EP23 TEACH · EP24 JPEG · EP25 INJECT · EP26 RECAP
