# RAG AI Teaching Assistant — Project README


## What this project does

This repository implements a small RAG-style assistant over video course content:

* Convert tutorial videos to MP3s (audio).
* Transcribe audio into JSON subtitle chunks using Whisper (`mp3_to_json.py`).
* Create embeddings for each subtitle chunk using a local embedding API (Ollama in this project) and save a dataframe `embeddings.joblib` (`preprocess_json.py`).
* Query the embeddings to find the most relevant chunks and build a prompt for a local LLM to answer course-related queries (`process_incoming.py`).

This README was generated from the original project README and the code files in the repo. See the original README for the high-level steps. fileciteturn0file5

Files used to generate this README (key scripts):

* `video_to_mp3.py` — converts videos -> mp3. fileciteturn0file7
* `mp3_to_json.py` — transcribes mp3 -> json using Whisper. fileciteturn0file1
* `preprocess_json.py` — creates embeddings and saves `embeddings.joblib`. fileciteturn0file2
* `process_incoming.py` — runs similarity search and calls the LLM for final answer; writes `prompt.txt` and `response.txt`. fileciteturn0file3
* Example prompt written to `prompt.txt`. fileciteturn0file4
* Resulting vector store saved as `embeddings.joblib`. fileciteturn0file0

---

## System requirements

* **OS:** Linux / macOS / Windows (WSL recommended for Windows)
* **Python:** 3.10 or newer
* **Disk:** Enough disk space for video/audio files and model downloads (Whisper `large-v2` model can be large — tens of GBs if using some backends).
* **Memory / CPU / GPU:** CPU-only will work but transcription and embedding generation are much faster with a GPU and CUDA configured. If you plan to use Whisper `large-v2` on CPU expect long processing times.
* **External tools:** `ffmpeg` must be installed and available on PATH (used by `video_to_mp3.py`).

---

## Python dependencies

Create a virtual environment and install dependencies (example):

```bash
python -m venv .venv
source .venv/bin/activate   # on Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

If you don't have a `requirements.txt`, install the main packages used in scripts:

```bash
pip install openai-whisper pandas numpy scikit-learn joblib requests
```

Notes:

* The code imports `whisper` (OpenAI Whisper). Installing `openai-whisper` or the appropriate fork is required.
* `scikit-learn` is used for `cosine_similarity`.
* `joblib` used to store `embeddings.joblib`.

---

## Services / APIs the project expects

* **Local Ollama (or another embedding/LLM API)** running at `http://localhost:11434`.

  * `preprocess_json.py` calls `/api/embed` with model `bge-m3` to compute embeddings. Ensure the embedding model (`bge-m3`) is available and Ollama is running. See the script for the exact request payload. fileciteturn0file2
  * `process_incoming.py` calls `/api/generate` with model `llama3.2` for the final answer. Ensure the LLM model is present in your Ollama server. fileciteturn0file3

If you don’t want to use Ollama, adapt the scripts to call another embeddings/LLM provider or replace the embedding call with an available embeddings service.

---

## Folder structure (expected)

```
project-root/
├─ videos/           # input tutorial videos (video_to_mp3.py reads this)
├─ audios/           # output mp3 files (video_to_mp3.py -> audios/)
├─ jsons/            # transcription jsons (mp3_to_json.py -> jsons/)
├─ embeddings.joblib  # saved dataframe with embeddings (preprocess_json.py)
├─ prompt.txt         # generated prompt for LLM (process_incoming.py)
├─ response.txt       # model's answer (process_incoming.py)
├─ video_to_mp3.py    # converts videos to mp3. fileciteturn0file7
├─ mp3_to_json.py     # whisper transcription. fileciteturn0file1
├─ preprocess_json.py # embeddings generation. fileciteturn0file2
├─ process_incoming.py# similarity search + LLM prompt generation. fileciteturn0file3
└─ README.md (this file)
```

---

## Step-by-step: How to run the pipeline

1. **Place videos** in `videos/` folder. Filenames should follow the pattern expected in `video_to_mp3.py` (script splits file names to extract tutorial number and title). fileciteturn0file7

2. **Convert videos -> mp3**

```bash
python video_to_mp3.py
```

This creates `.mp3` files in `audios/` named like `{tutorial_number}_{file_name}.mp3`. Requires `ffmpeg`. fileciteturn0file7

3. **Transcribe mp3 -> json** (Whisper)

```bash
python mp3_to_json.py
```

* This script loads Whisper (`large-v2`) and writes transcription JSON files to `jsons/` with chunk-level timing and text. Adjust the `language`/`task` arguments in the script as needed. fileciteturn0file1

4. **Create embeddings & build vector store**

```bash
python preprocess_json.py
```

* It reads all JSON files in `jsons/`, calls the local embed API (`/api/embed`), attaches `chunk_id` and `embedding` to each chunk, builds a `pandas.DataFrame` and saves it to `embeddings.joblib`. The file `embeddings.joblib` is used later for retrieval. fileciteturn0file2

5. **Ask a question / run the assistant**

```bash
python process_incoming.py
```

* This script: prompts the user for an incoming query, creates an embedding for that query, computes cosine similarities with the saved embeddings, selects top N chunks, generates a structured prompt (saved to `prompt.txt`) and calls the LLM with `/api/generate`. The result is printed and saved to `response.txt`. fileciteturn0file3

---

## Important configuration notes & tips

* **Whisper model size:** `large-v2` is used in the shipped script. If disk/compute are constrained, switch to a smaller model (e.g., `base`, `small`) in `mp3_to_json.py`. fileciteturn0file1
* **Ollama models:** Make sure `bge-m3` (embeddings) and `llama3.2` (generation) are installed/available locally. If your Ollama host or port differs, update the `requests.post` endpoints in `preprocess_json.py` and `process_incoming.py`. fileciteturn0file2fileciteturn0file3
* **Embeddings shape:** The embedding vectors are saved as lists in `embeddings.joblib`. `process_incoming.py` stacks them with `np.vstack` for similarity computation. If you change the embedding provider, ensure the output shape is consistent. fileciteturn0file3
* **Performance:** For large numbers of videos, consider batching embedding requests and saving intermediate results to avoid re-computing everything when a run fails. The current `preprocess_json.py` creates all embeddings in one pass.

---

## Troubleshooting

* **"ffmpeg not found"** — install ffmpeg and ensure it is in your PATH.
* **Whisper errors or model download failures** — check disk space and network. Use a smaller model if needed.
* **Ollama connection refused** — ensure Ollama is running on `localhost:11434` and your chosen models are loaded.
* **Slow embedding generation** — enable GPU for Ollama or use a smaller embedding model.

---

## Useful commands (quick)

```bash
# create venv & install
python -m venv .venv
source .venv/bin/activate
pip install openai-whisper pandas numpy scikit-learn joblib requests

# run pipeline
python video_to_mp3.py
python mp3_to_json.py
python preprocess_json.py
python process_incoming.py
```

---

## Where to look in the code

* `video_to_mp3.py` — audio extraction logic and ffmpeg call. fileciteturn0file7
* `mp3_to_json.py` — Whisper transcription: `model = whisper.load_model('large-v2')`, `model.transcribe(...)`. Adjust `language` or `task` here. fileciteturn0file1
* `preprocess_json.py` — embedding creation and saving `embeddings.joblib`. fileciteturn0file2
* `process_incoming.py` — input prompt handling, similarity search, prompt creation saved to `prompt.txt` and inference call. fileciteturn0file3

---

## Next improvements (suggested)

* Add a `requirements.txt` and `Makefile` or simple CLI wrapper for the whole pipeline.
* Add batching and resume capability to `preprocess_json.py` so large jobs can be resumed.
* Add unit tests for parsing filename structure and embedding shapes.
* Add a small web UI to ask queries instead of CLI.

---





