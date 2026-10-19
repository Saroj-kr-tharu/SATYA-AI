<div align="center">

```
  ███████╗ █████╗ ████████╗██╗   ██╗ █████╗
  ██╔════╝██╔══██╗╚══██╔══╝╚██╗ ██╔╝██╔══██╗
  ███████╗███████║   ██║    ╚████╔╝ ███████║
  ╚════██║██╔══██║   ██║     ╚██╔╝  ██╔══██║
  ███████║██║  ██║   ██║      ██║   ██║  ██║
  ╚══════╝╚═╝  ╚═╝   ╚═╝      ╚═╝   ╚═╝  ╚═╝

         ॥ सत्यमेव जयते ॥
         Truth always wins.
```

# Satya AI

**A multi-model AI arena: ask once, get answers from OpenAI, Claude and Gemini, and let an impartial judge pick the best one.**

![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-SDK-412991?logo=openai&logoColor=white)
![Anthropic](https://img.shields.io/badge/Anthropic-SDK-D97757)
![Gemini](https://img.shields.io/badge/Google-Gemini%20SDK-4285F4?logo=google&logoColor=white)
![Zod](https://img.shields.io/badge/Zod-Structured%20Output-3068B7)
![License](https://img.shields.io/badge/License-MIT-green)

</div>

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [How Satya AI Works](#how-satya-ai-works)
- [Concurrency: Calling 3 APIs at Once with `Promise.allSettled`](#concurrency-calling-3-apis-at-once-with-promiseallsettled)
- [Structured Output with Zod](#structured-output-with-zod)
- [Step-by-Step Reasoning Pipeline](#step-by-step-reasoning-pipeline)
- [The Judge Model](#the-judge-model)
- [Safety and Prompt-Injection Handling](#safety-and-prompt-injection-handling)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Sample Terminal Output](#sample-terminal-output)
- [Project Structure](#project-structure)
- [Known Issues and Roadmap](#known-issues-and-roadmap)
- [Tech Stack](#tech-stack)
- [License](#license)

---

## Overview

Different LLMs give different answers to the same question. **Satya AI** (*Satya* means "truth" in Sanskrit) sends your question to three leading models **in parallel**, forces each of them to reason through a fixed step-by-step pipeline with **structured JSON output**, and then calls a **judge model** that scores every answer on clear criteria and declares a winner with a reason.

| Role | Provider | SDK |
|------|----------|-----|
| Contestant A | Anthropic Claude (`claude-haiku-5-5`) | `@anthropic-ai/sdk` |
| Contestant B | OpenAI (`gpt-4o-mini`) | `openai` |
| Contestant C | Google Gemini (`gemini-3.8-flash`) | `@google/genai` |
| Judge | Anthropic Claude (`claude-haiku-5-5`) | `@anthropic-ai/sdk` |

## Key Features

- **Multi-model comparison**: one question, three independent answers.
- **Parallel execution**: all three API calls run at the same time using `Promise.allSettled`.
- **Fault tolerance**: if one provider fails (quota, network, auth), the others still complete and the judge scores the failure as `0`.
- **Structured output**: Zod schemas enforce a strict JSON shape for every model response and for the judge verdict.
- **Step-by-step reasoning**: every model follows the same `initial → think → analyze → output` pipeline.
- **Impartial AI judge**: scores each answer on relevance, accuracy, clarity and completeness (0-10 each, 40 total).
- **Prompt-injection hardening**: user input is treated as untrusted data in all prompts.
- **Readable terminal UI**: summary table, word-wrapped pipelines, score table and score bars.

## Architecture

```mermaid
flowchart TD
    U[User Question] --> W["Wrap in &lt;user_question&gt; tags"]
    W --> P{{"Promise.allSettled"}}
    P --> C[Claude SDK]
    P --> G[OpenAI SDK]
    P --> M[Gemini SDK]
    C --> R[Collect Results]
    G --> R
    M --> R
    R -->|"failed = No response received."| J["Judge AI (Claude)<br/>Answers labelled A, B, C"]
    J --> S["Scores + Winner + Reason"]
    S --> T[Terminal Report]
```

## How Satya AI Works

1. **Input**: you pass a question to `aimodel.AskQuestion("...")`.
2. **Sanitising**: the question is wrapped in `<user_question>` tags and every system prompt declares that all user text is untrusted data.
3. **Parallel calls**: Claude, ChatGPT and Gemini are called simultaneously, each with the same system prompt and the same structured-output schema.
4. **Reasoning pipeline**: each model returns four ordered steps (`initial`, `think`, `analyze`, `output`) as JSON.
5. **Aggregation**: results are collected with `Promise.allSettled`. A rejected promise becomes `{ error: "..." }` instead of crashing the program.
6. **Judging**: the final `output` of each model is placed into `<answer_A>`, `<answer_B>`, `<answer_C>` tags and sent to the judge. Missing or failed answers are replaced with `"No response received."`.
7. **Verdict**: the judge runs its own 4-step pipeline, returns scores, a winner and a reason, and the terminal prints a formatted report.

## Concurrency: Calling 3 APIs at Once with `Promise.allSettled`

Node.js is single-threaded, but network I/O is **non-blocking**. When you `await` an HTTP request, the event loop is free to start other requests while waiting for responses. That means three API calls can be *in flight at the same time* without any threads.

```js
const [chatgpt, claude, gemini] = await Promise.allSettled([
  this.Chatgpt(Prompt),
  this.ClaudeAi(Prompt),
  this.GeminiAi(Prompt),
]);
```

**Why `allSettled` and not `Promise.all`?**

| | `Promise.all` | `Promise.allSettled` |
|---|---|---|
| Waits for | All to succeed | All to finish (success or failure) |
| If one rejects | Rejects immediately, other results are lost | Returns every result with a status |
| Result shape | Array of values | Array of `{ status, value }` or `{ status, reason }` |
| Fit for Satya AI | Poor, one quota error kills everything | Ideal, partial results are still useful |

Each settled result is then normalised:

```js
claude.status === "fulfilled" ? claude.value : { error: claude.reason?.message }
```

**Time cost:** total latency is roughly the *slowest* provider, not the sum of all three.

```
Sequential : |--Claude--|--OpenAI--|--Gemini--|        ≈ 3x
Parallel   : |--Claude--|
             |--OpenAI--|                               ≈ 1x (slowest)
             |--Gemini--|
```

## Structured Output with Zod

Instead of hoping the model returns valid JSON, Satya AI defines the shape once with **Zod** and passes it to each SDK's native structured-output feature.

```js
PipelineStep = z.object({
  step: z.enum(["initial", "think", "analyze", "output"]),
  content: z.string(),
});

ModelFormat = z.object({
  input: z.string(),
  pipeline: z.array(this.PipelineStep).length(4),
  output: z.string(),
});
```

| Provider | Structured-output helper |
|----------|--------------------------|
| OpenAI | `zodTextFormat(schema, name)` via the Responses API |
| Claude | `zodOutputFormat(schema)` via `output_config.format` |
| Gemini | `zodToJsonSchema(schema)` via `responseJsonSchema` |

The judge has its own schema (`JudgeFormat`) containing `scores` for A/B/C, a `winner` enum, a `reason` and a final `output`.

## Step-by-Step Reasoning Pipeline

Every contestant must reason in exactly four ordered steps:

| # | Step | Purpose |
|---|------|---------|
| 1 | `initial` | State what the user is trying to do |
| 2 | `think` | Break the problem down and plan |
| 3 | `analyze` | Solve step by step and verify the result |
| 4 | `output` | Give the final answer |

The system prompt includes six few-shot examples (percentages, prompt injection, "educational" jailbreak framing, harmful requests, BODMAS maths, average speed) so the models follow the format and refusal behaviour consistently. Refused requests skip the detailed analysis and return a short, polite refusal.

## The Judge Model

The judge receives the question and anonymised answers labelled **A, B, C**, then scores each one from 0 to 10 on four criteria:

| Criterion | Question it answers |
|-----------|---------------------|
| **Relevance** | Does it directly answer the question asked? |
| **Accuracy** | Is the information factually correct? |
| **Clarity** | Is it easy to read and understand? |
| **Completeness** | Does it fully cover what was asked? |

**Total = 0 to 40 per answer.**

Judging rules:

- Strictly impartial; no preference for any model or writing style.
- Longer is not better; a short, correct answer beats a long, vague one.
- Confident but wrong answers are penalised heavily on accuracy.
- Wrong language reduces the clarity score.
- Missing or null answers score `0` on every dimension.
- Ties go to the most concise answer.
- Formatting alone never decides the winner.

The judge returns its own `initial → think → analyze → output` pipeline, the scores, the `winner`, a `reason` and a final verdict. The response is validated (`winner`, `scores` and `reason` must exist) before it is printed.

## Safety and Prompt-Injection Handling

- All user text is treated as **untrusted data**, with or without the `<user_question>` tags.
- Requests to ignore rules, reveal the system prompt, switch roles or act as another AI are refused.
- The judge also treats `<question>` and `<answer_X>` content as data, so a model's answer cannot hijack the judge.
- Political topics: neutral, factual summaries only; no personal opinions.
- Illegal or harmful requests are declined with a safe, legal alternative where possible.

## Installation

**Prerequisites**

- Node.js 18 or newer
- API keys for Anthropic, OpenAI and Google AI Studio (Gemini)

```bash
git clone https://github.com/<your-username>/satya-ai.git
cd satya-ai
npm install @anthropic-ai/sdk openai @google/genai dotenv zod
```

Enable ES modules in `package.json` (the code uses `import`):

```json
{
  "name": "satya-ai",
  "version": "1.0.0",
  "type": "module"
}
```

## Configuration

Create a `.env` file in the project root:

```env
ANTHROPIC_API_KEY=your_anthropic_key
OPENAI_API_KEY=your_openai_key
GEMINI_API_KEY=your_gemini_key
```

> Never commit `.env`. Add it to `.gitignore`.

## Usage

Edit the question at the bottom of `index.js`:

```js
const aimodel = new aiModel();

aimodel.AskQuestion(
  "What is the difference between var, let and const in JavaScript? Give a real example where using the wrong one causes a bug."
);
```

Run it:

```bash
node index.js
```

You can also test a single provider while debugging:

```js
aimodel.Chatgpt("what is 2 + 5");
aimodel.ClaudeAi("what is 2 + 5");
aimodel.GeminiAi("what is 2 + 5");
```

### Public methods

| Method | Description |
|--------|-------------|
| `AskQuestion(prompt)` | Full flow: parallel calls, results report, judge, verdict |
| `Chatgpt(prompt)` | Calls OpenAI with structured output |
| `ClaudeAi(prompt)` | Calls Claude with structured output |
| `GeminiAi(prompt)` | Calls Gemini with structured output |
| `JudgeAi(input)` | Scores answers and picks the winner |
| `PrintResults(results)` | Prints the summary table and pipelines |
| `printJudgeResult(response)` | Prints scores, bars, winner and verdict |

## Sample Terminal Output

Question: *"What is the difference between var, let and const in JavaScript? Give a real example where using the wrong one causes a bug."*

```text
◇ injected env (3) from .env
<------------- CALLING CHATGPT -------------->
<------------- CALLING Claude AI -------------->
<------------- CALLING GeminiAI -------------->

════════════════════════════════════════════════════════════════════════════════
                    📊 MULTI AI ANSWER JUDGE - RESULTS
════════════════════════════════════════════════════════════════════════════════

📋 SUMMARY TABLE
────────────────────────────────────────────────────────────────────────────────
  MODEL       STATUS      STEPS     OUTPUT PREVIEW
────────────────────────────────────────────────────────────────────────────────
  CLAUDE      ✅ OK        4         ...
  CHATGPT     ✅ OK        4         The difference is that 'var' is function...
  GEMINI      ❌ FAILED    ─         Quota / API Error
────────────────────────────────────────────────────────────────────────────────

  🔄 PIPELINE (Claude)
  🔵 [1] INITIAL   The user wants to understand the differences between var,
                   let, and const in JavaScript and see a real bug...
  🟡 [2] THINK     var is function-scoped and hoisted; let is block-scoped;
                   const cannot be reassigned...
  🟠 [3] ANALYZE   for (var i = 0; i < 3; i++) { setTimeout(() =>
                   console.log(i), 0); } prints 3, 3, 3 ...
  🟢 [4] OUTPUT    var is function-scoped, let and const are block-scoped...

⚖️  Calling Judge AI...

════════════════════════════════════════════════════════════════════════════════
⚖️   FINAL JUDGE RESULT
════════════════════════════════════════════════════════════════════════════════

📊 SCORES
────────────────────────────────────────────────────────────────────────────────
  LABEL MODEL       RELEVANCE   ACCURACY    CLARITY     COMPLETE    TOTAL
    A   🟢 ChatGPT  0/10        0/10        0/10        0/10        0/40
 🏆 B   🟠 Claude   6/10        5/10        7/10        3/10        21/40
    C   🔵 Gemini   0/10        0/10        0/10        0/10        0/40

📈 SCORE BAR
  🟢 ChatGPT   [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 0/40
  🟠 Claude    [████████████████░░░░░░░░░░░░░░] 21/40 🏆
  🔵 Gemini    [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 0/40

🏆 WINNER
  🟠  Answer B  →  CLAUDE

✅ FINAL VERDICT
  The best answer is B because it is the only substantive response, correctly
  explaining var's function/global scope versus let and const's block scope.
  It loses points for misleadingly calling const 'immutable' and for not
  providing the real bug example the question requested.
```

This run also shows the fault tolerance in action: Gemini hit its free-tier quota, yet the program finished and the judge scored the missing answer as `0`.

## Project Structure

```text
satya-ai/
├── index.js        # aiModel class: prompts, schemas, provider calls, judge, printers
├── .env            # API keys (not committed)
├── package.json
└── README.md
```

## Known Issues and Roadmap

These come from reading the code against the sample run above and are worth fixing before a public release:

- [ ] **Judge label mapping is swapped.** `AskQuestion` sends Claude as **A**, ChatGPT as **B**, Gemini as **C**, but `printJudgeResult` maps `A → ChatGPT`, `B → Claude`. The winner and score table can therefore show the wrong model name.
- [ ] **`ClaudeAi` returns `steps[0]`.** With structured output the first response is already the full `{ input, pipeline, output }` object, so the loop is unnecessary. In the sample run Claude's `output` came back empty, which the judge then scored as missing.
- [ ] **"Final Answer Comparison" prints `undefined`.** `printInWrap` prints and returns nothing, so it cannot be used inside a template string.
- [ ] **`GeminiAi` returns an array** (`steps`) while the other providers return an object, so Gemini results break `PrintResults` and the judge input when it succeeds.
- [ ] **Judge is not schema-enforced.** `JudgeFormat` is defined but not passed to `JudgeAi`; the response relies on prompt instructions and `JSON.parse` only.
- [ ] Typos in prompts (`intial`, `recevie`, `sequeenze`) should be corrected, and the judge step name must be `initial` to match the schema.
- [ ] Accept the question from the CLI (`node index.js "your question"`).
- [ ] Add retry with backoff for rate-limit errors.
- [ ] Export results to JSON or Markdown.
- [ ] Optional web UI.

## Tech Stack

- **Runtime:** Node.js (ES modules)
- **LLM SDKs:** `openai`, `@anthropic-ai/sdk`, `@google/genai`
- **Validation and schemas:** `zod`
- **Config:** `dotenv`

## License

Released under the [MIT License](LICENSE).

<div align="center">

**॥ सत्यमेव जयते ॥**

*Built with curiosity, to let the best answer win.*

</div>