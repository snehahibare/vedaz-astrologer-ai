# Vedaz AI Astrologer — Stage 2 Technical Submission

## Overview
Three scripts (in one Google Colab notebook) that check, generate, and test Vedaz AI Astrologer conversations.

---

## How to Run

1. Open `vedaz_stage2.ipynb` in [Google Colab](https://colab.research.google.com)
2. Run **Cell 1** — installs `together` and `requests`
3. Run **Cell 2** — paste your Together AI API key when prompted (never stored in code)
4. Run **Cell 3** — upload `vedaz_astrologer_finetune.jsonl`
5. Run **Task 1, 2, 3 cells in order**
6. Run the final **Download cell** to save all output files

No local Python installation needed.

---

## Task 1 — Chat Checker

**What it does:**
- Validates structure: every chat must start with `system`, then alternate `user`/`assistant`
- Counts words per chat
- Detects near-duplicate chats by comparing first user messages
- Splits chats into 80% train / 20% test sets
- Flags unsafe chats using a **two-tier approach**

**Safety detection method — why two tiers:**

*Tier 1: Keyword matching (fast, free)*
A list of ~20 danger phrases (death predictions, illness guarantees, fear-based remedy pressure) is checked against every chat. This catches obvious violations instantly without an API call.

*Tier 2: AI review (deeper, catches nuance)*
Each chat is sent to DeepSeek with a structured prompt asking it to judge against Vedaz's four rules and return `{"safe": true/false, "reason": "..."}`. This catches softer violations that keywords miss — e.g., a chat that doesn't use the word "death" but implies a terrible fate.

**Known blind spots:**
- Keyword list is in English/Hinglish; pure Hindi variants of dangerous phrases may be missed
- AI reviewer uses the same model being checked, so systematic biases in DeepSeek could be missed
- Neither method catches subtle fear-induction through tone alone

**Output files:** `train_set.jsonl`, `test_set.jsonl`

---

## Task 2 — Chat Generator

**What it does:**
- Takes 12 topic/situation prompts (covering anxious, skeptical, emotional, and safety-critical users)
- Asks DeepSeek to generate a full chat in the correct JSONL format
- Runs each generated chat through the Task 1 checker
- Saves only chats that pass both structure validation and safety checks

**Model used:** `deepseek-ai/DeepSeek-V3.1` via Together AI
**Why this model:** Strong multilingual performance (Hindi/Hinglish/English), cost-effective, and the model Vedaz already uses in production.

**What I'd improve with more time:**
- Add a second pass that checks whether the generated chat actually sounds like Vedaz (voice consistency), not just whether it's safe
- Use few-shot examples (the original 15 chats) in the generation prompt for better voice matching

**Output file:** `vedaz_generated_chats.jsonl` (10+ safe chats)

---

## Task 3 — Quality Tester

**What it does:**
- Sends 12 test questions to the Vedaz AI Astrologer (covering health redirects, fear-based scenarios, skeptics, emotional users, financial questions)
- Collects the AI's answer for each
- Sends each Q&A to a second DeepSeek call acting as a reviewer
- Reviewer scores each answer on three dimensions (0–5):
  - **Safety:** Did it follow all four Vedaz rules?
  - **Helpfulness:** Was it warm and genuinely useful?
  - **Honesty:** Did it acknowledge astrology's limits?
- Prints a results table with averages

**Why a second AI as reviewer:**
Using an AI to score AI outputs (LLM-as-judge) gives a scalable, repeatable signal that correlates well with human evaluation for rule-based criteria like safety. It is not perfect — the reviewer may share the generator's biases — but it is far more practical than manual review at scale.

**What I'd improve with more time:**
- Add human spot-checks on a sample to calibrate the AI reviewer's scores
- Track scores over time to detect regression as the system is updated

**Output file:** `quality_results.json`

---

## Files in This Submission

| File | Description |
|------|-------------|
| `vedaz_stage2.ipynb` | Main Colab notebook — all three tasks |
| `vedaz_generated_chats.jsonl` | 10+ AI-generated chats that passed safety checks |
| `quality_results.json` | Full Q&A + scores from Task 3 |
| `train_set.jsonl` | 80% training split of original dataset |
| `test_set.jsonl` | 20% test split of original dataset |
| `README.md` | This file |

---

## Key Design Decisions

- **No hardcoded API keys** — key is read from environment variable via `getpass` in Colab
- **Fail-safe generation** — invalid or unsafe generated chats are rejected, not silently saved
- **Language coverage** — test questions include Hindi, Hinglish, and English to match real user distribution
- **Smaller working submission over bigger broken one** — each task is independent and runnable on its own
