# AI Gameplay Analysis System

An end-to-end AI pipeline that extracts player behavior, tendencies, and evidence-grounded insights from raw gameplay footage — no game APIs, no stat exports, no post-game summaries.

Built as an AI systems engineering project. The core design problem was making two specialized models work together reliably under real consumer hardware constraints, with every output traceable back to timestamped visual evidence.

---

## How It Works

The system separates perception from reasoning across two stages:

**Interpreter** — a vision-language model that processes gameplay video in overlapping batches, reads HUD elements and player actions, and produces timestamped narration of what happened. Answers: *what happened?*

**Reasoner** — an LLM that consumes the interpreter's output, identifies recurring patterns and tendencies across the match, compares against historical sessions, and produces evidence-grounded recommendations. Answers: *what does it mean?*

Every claim the reasoner makes is required to cite a specific timestamp from the interpreter's output. If it cannot, it does not make the claim.

---

## Architecture

```
Gameplay Video
      ↓
Frame Extraction & Preprocessing
      ↓
Interpreter — Vision-Language Model (Qwen3-VL)
      ↓
Timestamped Narration (JSON)
      ↓
Reasoner — Analysis Model (Qwen3-14B)
      ↓
Player Profile + Recommendations
```

The two models are intentionally separate. A single multimodal model capable of both tasks requires VRAM far beyond consumer hardware. Separation also improves explainability and makes each stage independently debuggable.

---

## Key Engineering Decisions

**Overlapping batch processing**
Running VLM inference against decoded video frames hits VRAM limits on a single pass. The system processes frames in overlapping batches — each batch receives both a structured JSON context summary and raw frames from the prior batch. Context alone caused hallucinations in testing because the model had no footage to verify it against. Overlapping frames provide the visual evidence that keeps each batch grounded.

**Structured JSON inter-batch context**
Context passed between batches is encoded as structured JSON — enough to preserve continuity without consuming the VRAM budget reserved for inference. Models follow structured formats more reliably than free-form text, which reduced context-boundary errors significantly.

**Citation-enforced reasoning**
The reasoner is prompted to cite a specific interpreter timestamp before stating any claim. This was the solution to a hallucination problem where the model invented gameplay events, patterns, and in-game elements that never appeared in the footage. Requiring citations against a fixed source eliminated fabrication.

**Two-model selection**
Models were selected by benchmarking VLMs and LLMs separately against the specific demands of each stage — visual extraction vs. pattern reasoning — and choosing quantized variants small enough to inference within a 16GB VRAM budget. Qwen3-VL handles the visual stage; Qwen3-14B handles reasoning.

---

## Reasoner Constraints

The reasoner is explicitly instructed to:

- Cite timestamped evidence from the interpreter before any claim
- Focus only on player-controlled decisions
- Avoid motivational or coaching-companion tone
- Avoid inventing mechanics, scorestreaks, or game elements
- Avoid team coordination advice

---

## Requirements

- Python 3.12.10
- Ollama
- FFmpeg
- GPU with 16GB+ VRAM (RTX 5080 or equivalent)
- Models:
  - `qwen3-vl:8b-instruct-q4_K_M` — visual interpretation
  - `qwen3:14b-q4_K_M` — behavioral reasoning

---

## Setup

```bash
pip install -r requirements.txt

ollama pull qwen3-vl:8b-instruct-q4_K_M
ollama pull qwen3:14b-q4_K_M

# Place gameplay videos in videos/
```

---

## Usage

```bash
# Full pipeline with interactive coaching
python main.py analyze videos/match.mp4 --map Nuketown --mode Domination

# Batch mode (non-interactive)
python main.py analyze videos/match.mp4 --map Nuketown --mode Domination --batch

# Interpreter only — video to narration JSON
python main.py interpret videos/match.mp4 --map Nuketown --mode Domination

# System health check
python main.py check --all

# Profile management
python main.py profile show
python main.py profile create --player-id myname
```

**Common options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--quality` | `high` | Processing quality: `high` or `fast` |
| `--fps` | `1` | Frames per second to extract |
| `--overlap` | `5` | Overlap frames between batches |
| `--batch` | off | Non-interactive batch mode |
| `--output-dir` | `.` | Output directory |

---

## Docker Deployment

Two containers: `ollama` for model serving, `gameplay` for the analysis pipeline.

```bash
# First-time setup
docker compose up -d ollama
docker compose --profile init up ollama-init

# Run analysis
docker compose run --rm gameplay analyze /app/videos/match.mp4 \
  --map Nuketown --mode Domination --batch

# System check
docker compose run --rm gameplay check --all
```

See [DOCKER.md](DOCKER.md) for full deployment documentation.

---

## Limitations

- Not tested on matches longer than 10 minutes
- Requires clear HUD visibility throughout
- VLM may misinterpret ambiguous or fast-moving visuals
- Requires 16GB+ VRAM for full quality mode
- Assumes Ollama is installed and running locally

---

## Generalizability

Currently implemented for Call of Duty. The architecture is game-agnostic — adapting to a different game requires updating the interpreter and reasoner prompts to match the new HUD layout and game mechanics.

---

## Tech Stack

Python · Qwen3-VL · Qwen3-14B · Ollama · FFmpeg · Docker
