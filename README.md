# AI Resume-Grounded Interview Platform

A proctored mock-interview platform that interviews you about **your own resume** and then tests your **core CS fundamentals**, the way a real placement interview does, and explains every score. All models run **locally**: no paid API and no GPU are needed at interview time.

**Stack:** Python 3.12 · Flask · SQLAlchemy (SQLite / PostgreSQL) · Redis · Docker Compose · PyTorch · Hugging Face Transformers · sentence-transformers · DeBERTa-v3 · NLI cross-encoders · Whisper (faster-whisper) · FAISS + BM25 · Qwen3-8B (Ollama, offline) · MediaPipe · vanilla JS

---

## What it does

- **Full Interview, two proctored rounds**
  1. **Round 1 · Resume Discussion:** 10 questions, each traceable to a specific project, job, certification or skill on the candidate's resume, scored across named dimensions with strengths, gaps and coaching.
  2. **Round 2 · Technical:** 8 questions across **Computer Networks, DBMS, DSA, OOAD and Operating Systems**, with a follow-up or clarification request when an answer misses a key point or is unclear.
  3. A combined report, saved to the candidate's history.
- **Technical Interview:** the technical round on its own (10 questions).
- **Resume Intelligence Engine:** a local, deterministic 8-stage parser (PDF/DOCX/TXT → structured Candidate Profile).
- **Proctoring in the browser:** camera setup check, gaze and head-pose attention, head-turn liveness checks, second-person and low-light/blur alerts, and zero-tolerance full-screen / no-tab-switch / no-copy-paste rules. Webcam frames never leave the browser.
- **Voice answers in any browser:** click the mic to record, click again to stop; the server turns the recording into text with Whisper (CPU, local), so it works in Chrome, Edge, Brave, Firefox and Safari and no audio is sent to a third-party speech service.
- **Accounts and progress:** sign-up with academic details, resume and profile photo; every session saved turn by turn; Home stats, score trend and a "Focus next" list of recurring weaknesses.
- **Production mode with one command:** `docker compose up` runs the app with PostgreSQL and Redis. Interviews in progress survive an app restart, and several app servers can share the same users.

## Measured results

| Component | Result |
|---|---|
| Slide retrieval (new pipeline vs old page-per-chunk index) | hit@5 **96–100%** vs 68–82% on hand-labelled questions |
| Question bank | **422** questions generated from ~5,800 lecture slides; every key point backed by a verified slide quote; 156 hand-reviewed top-tier questions served |
| Round 1 answer evaluator (blended DeBERTa-v3 + heuristic) | QWK **0.86** on a 151-answer held-out set (heuristic alone: 0.10); 0.70 on a 58-answer stress set |
| Round 2 technical grader | Same move-on decision as a human: 92% on 400 LLM-written answers, **66% on fresh human-written answers** (up from 64%); score correlation with humans **0.44 → 0.55** and confidently-wrong answers **0.42 → 0.29** after the grader work below (fresh set, never used for tuning) |
| Performance | Server start 6.8 s → **2.1 s**; ~0.3 s per answer on CPU |
| Voice answers (Whisper `base.en`, CPU) | An 11 s technical answer transcribed word-perfect ("Redis", "Docker", "800 milliseconds") in **0.6 s** natively, ~2 s in Docker; `small.en` was 3× slower and misheard "Redis" |
| Resilience (PostgreSQL + Redis) | Two app servers sharing one database and one Redis ran an interview with every request alternating between them, and one restarted mid-interview: the interview carried on. Same for the Docker app container restarted mid-round |
| Tests | 1,267 app tests (also run on PostgreSQL 16 and on Linux in the container), 511 resume-engine tests, 19 slide-pipeline tests, 10 frontend test files, plus a grader regression benchmark |

## My contributions

This was a 4-person capstone project at PES University. The work below is mine:

- **Slide RAG pipeline** (`rag_system/rag_tester/slide_rag/`): layout-aware slide extraction, running-header/logo removal, slide classification, section grouping, OCR and a vision model for diagram slides, and hybrid FAISS + BM25 retrieval with a cross-encoder reranker. Built and evaluated with hand-labelled gold sets and ablations.
- **Technical question bank:** offline generation with local Qwen3-8B, in-code quote verification against the slides, an LLM judge, confidence ranking, follow-up enrichment, hand review of every active question, key-point repair, and interview-frequency tiers so the most-asked topics come up most.
- **Technical interview engine** (`main_cap/cap/tech_interview/`): question selection (distinct topics, difficulty arc, no repeats, weak-topic weighting), the NLI key-point grader, follow-up and clarification policy, and the live session.
- **Grader reliability work:** a regression benchmark against ~630 human-labelled answers, including a fresh set kept out of tuning; a contradiction check that caps fluent wrong answers; abbreviation expansion ("DFS" = "depth-first search"); a similarity floor against NLI hallucinations; and a no-label sanity check that every served question passes its own model answer.
- **Round 1 evaluation integration:** blended a fine-tuned DeBERTa-v3 model (0.8) with the heuristic evaluator (0.2), added a non-answer filter (0 of 628 real answers wrongly flagged) and data-driven end-of-session feedback.
- **Integration and product work:** integrated the improved resume engine and webcam features from teammates' branches (with fixes and tests); the two-round interview flow, profile photos, proctoring fixes, startup performance, documentation and deployment.
- **Production infrastructure:** PostgreSQL support (UTC-pinned connections, length-safe writes, tests that run on both databases), live interview state moved out of process memory into a pluggable store (memory or Redis) with deterministic Round 1 replay, a waitress server, a Docker Compose stack (CPU-only PyTorch, models baked into the image, non-root user, health checks, data in volumes), and `copy_database.py` to move every account, session and resume file between databases.
- **Voice input:** replaced the browser's speech API (Chrome/Edge only) with in-page recording and server-side Whisper transcription, with up-front camera/mic permission so the prompt never breaks full screen mid-interview.

---

## Quick start

**Requirements:** Python 3.12, ~8 GB RAM, a recent browser (mostly tested in Chrome and Edge) with a webcam and microphone, internet on the first run (model downloads). No GPU needed. Full list: [`SETUP_PREREQUISITES.md`](SETUP_PREREQUISITES.md).

```bash
pip install -r Requirements_Global.txt
cd main_cap/cap
python app.py          # http://localhost:5000
```

Sign up with a resume and start an interview from Home. The SQLite database and its migrations are created automatically.

**Trained Round 1 model weights are not included** (735 MB). Without them the app runs normally, and Round 1 uses the heuristic evaluator; the startup log says which evaluator is active.

### Production mode with Docker

One command runs the app with PostgreSQL and Redis, with no Python install. Needs [Docker Desktop](https://www.docker.com/products/docker-desktop/) (WSL 2 on Windows). From the repository root:

```bash
docker compose up --build -d     # first build ~15 min (downloads ~3 GB); later starts take seconds
docker compose logs -f app       # watch it start
docker compose down              # stop (data is kept in volumes; `down -v` deletes it)
```

| | `python app.py` (simple mode) | `docker compose up` (production mode) |
|---|---|---|
| Database | SQLite file | PostgreSQL 16 |
| Interviews in progress | Process memory; a restart ends them | Redis; survive a restart |
| Server | Flask development server | waitress (`serve.py`) |
| Install | Python 3.12 + packages | Docker only |

Optional settings (port, database password, secret key, threads): copy `.env.example` to `.env`. To move existing data from the SQLite file into the Docker database, see `copy_database.py` and [`PROJECT_DOCUMENTATION.md`](PROJECT_DOCUMENTATION.md) §3.6.

## Tests

```bash
cd main_cap/cap
python -m pytest --ignore=resume_engine/tests -q    # app
python -m pytest resume_engine/tests -q             # resume engine
for f in test_*.js; do node "$f"; done              # frontend

cd ../..
python grader_eval/benchmark.py                     # technical grader vs human labels (no regressions vs baseline)
python grader_eval/sanity_check.py                  # every served question passes its own model answer
```

## Repository layout

```
main_cap/cap/                 Flask app
  app.py, account_routes.py, session_routes.py     HTTP API, accounts, history
  serve.py, live_store.py, database.py, copy_database.py    Production server, live state (memory/Redis), database
  speech_to_text.py           Voice answers: Whisper transcription
  resume_engine/              8-stage resume parser
  planner.py, topic_pool.py, question_realizer.py, conversation_engine.py    Round 1
  averaged_evaluator.py, heuristic_evaluator.py, answer_gate.py, interview_feedback.py    Round 1 scoring
  tech_interview/             Round 2: bank, selector, NLI grader, follow-ups, session
  templates/index.html        Web UI, proctoring and webcam monitoring
rag_system/rag_tester/
  slide_rag/                  Offline slide pipeline and question-bank tooling
  question_bank/              The 422-question bank (one JSON per subject) + quality review + repairs
grader_eval/                  Labelled answer sets, benchmark, sanity check
resume_classifier/            Text extraction for DOCX/TXT resumes
Dockerfile, docker-compose.yml, docker/    Production stack (app + PostgreSQL + Redis)
```

## Documentation

- [`PROJECT_DOCUMENTATION.md`](PROJECT_DOCUMENTATION.md): architecture, every subsystem, design decisions, measured results and known limitations (detailed engineering notes).
- [`SETUP_PREREQUISITES.md`](SETUP_PREREQUISITES.md): what a new machine needs.
- [`rag_system/rag_tester/question_bank/quality_review.md`](rag_system/rag_tester/question_bank/quality_review.md): per-question quality review.
