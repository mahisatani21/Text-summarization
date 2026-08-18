# Execution Guide — 20 Records / 20 Optuna Trials (Windows PowerShell)

This is the exact code from `github.com/saumyapatel792/Text-summarization`, with **one fix applied**:

> **What changed:** In `src/zenml_pipeline.py`, the number of Optuna trials (`n_trials`) was hardcoded to `2` and never read the `N_TRIALS` environment variable — unlike the record counts, which already did. Two lines were added/changed so `N_TRIALS` now controls the ZenML pipeline too, the same way `BENCHMARK_SAMPLE_SIZE` already did. Nothing else in the codebase was touched.

Everything else (models, thresholds, Docker configs, dashboards) is identical to the original repo.

---

## 0. Before you start

- Unzip this folder **into** your existing project folder:
  `D:\Alliance University\Sem-3\Machine Learning Operations\project_august\Text-summarization`
  so that your existing `venv\` folder sits alongside the `src\`, `config\`, `docker\` folders from this zip.
- If you already have an old `mlflow.db` / `mlruns\` folder there from previous runs, decide whether to keep it (new runs just get added to the same experiment) or rename it to start fresh — either is fine.

---

## 1. Open PowerShell and activate your virtual environment

```powershell
cd "D:\Alliance University\Sem-3\Machine Learning Operations\project_august\Text-summarization"
venv\Scripts\activate
```

You should see `(venv)` appear at the start of your prompt.

Confirm the files are in place:
```powershell
dir src
```
You should see: `app.py`, `config.py`, `metrics.py`, `train.py`, `tune.py`, `utils.py`, `zenml_pipeline.py`.

---

## 2. Start the local MLflow tracking server

Open a **second, separate PowerShell window** (keep the first one for later commands).

```powershell
cd "D:\Alliance University\Sem-3\Machine Learning Operations\project_august\Text-summarization"
venv\Scripts\activate
venv\Scripts\mlflow server --host 127.0.0.1 --port 5000 --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns
```

Leave this window running for the rest of the guide. It's your MLflow UI backend.

📸 **Screenshot opportunity #1:** Once it prints `Listening at: http://127.0.0.1:5000`, open that URL in your browser — this confirms the tracking server is live before you run anything. Good "setup proof" slide, optional.

---

## 3. Run the standalone workflow: `train.py` → `tune.py`

Back in your **first** PowerShell window (venv activated):

```powershell
$env:BENCHMARK_SAMPLE_SIZE="20"
$env:TUNE_SAMPLE_SIZE="20"
$env:N_TRIALS="20"
$env:PYTHONPATH="."
$env:MLFLOW_TRACKING_URI="http://127.0.0.1:5000"
$env:PYTHONIOENCODING="utf-8"

python src/train.py
```

This benchmarks 3 candidate models (`distilbart-cnn-6-6`, `t5-small`, `flan-t5-small`) on 20 records each and logs 3 runs to MLflow. It will print the best model name and ROUGE-L score at the end — this takes the longest of all steps since it's loading 3 separate models.

Then run tuning:
```powershell
python src/tune.py
```

This runs 20 Optuna trials (num_beams, length_penalty search) on 20 records, using the default model (`sshleifer/distilbart-cnn-6-6` unless you set `$env:BEST_MODEL_NAME` first). It logs 1 parent run + 20 nested trial runs, and registers the best model as `TextSummarizerBest` in the MLflow Model Registry.

⏱️ Expect this to take a while longer than your original 3 records/3 trials run — 20×20 = 400 inference calls on CPU.

---

## 4. Run the ZenML pipeline (the fixed version)

Same terminal, same env vars still active:

```powershell
python src/zenml_pipeline.py
```

This runs the full Ingest → Validate → Transform → Train (20 trials) → Evaluate → Deploy pipeline on 20 records, and — thanks to the fix — genuinely uses 20 Optuna trials this time instead of silently running only 2.

---

## 5. Verify everything landed in MLflow

In your browser, go to `http://127.0.0.1:5000`, click experiment **`Text_Summarization_System`**.

You should now see, added to whatever was there before:
- 3 new runs from `train.py` (`benchmark_*`)
- 1 parent + 20 nested trial runs from `tune.py` (`optuna_tuning_*`, `trial_0` … `trial_19`)
- 1 parent + 20 nested trial runs from `zenml_pipeline.py` (`zenml_tuning_*`, `zenml_trial_0` … `zenml_trial_19`), plus a `zenml_deploy` run if the ROUGE-L threshold (0.12) was met

📸 **Screenshot opportunity #2 (important for presentation):** The full runs table for `Text_Summarization_System`, showing the new 20-trial runs. This is your main "look how much more thorough the tuning was" evidence.

📸 **Screenshot opportunity #3 (important):** Select the 20 nested trials from either `tune.py` or `zenml_pipeline.py`, click **Compare**, scroll to **Parallel Coordinates Plot**, add `num_beams`, `length_penalty` as parameters and `rougeL` as the metric. With 20 trials instead of 2–3, this plot will actually show a meaningful spread/pattern — much more convincing on 20 points than on 2.

📸 **Screenshot opportunity #4:** Same comparison page → **Scatter Plot**, X-axis `length_penalty`, Y-axis `rougeL`. Again, far more informative with 20 data points.

📸 **Screenshot opportunity #5:** Go to the **Models** tab → `TextSummarizerBest` → latest version. Show the **Source Run** links back to the winning trial out of 20, and the **Schema**/**Artifacts** tabs. This proves the registry picked the best of a real 20-trial search, not a 2-trial one.

---

## 6. Export the updated CSV (optional but recommended for your report)

```powershell
python mlflow_export.py
```

This regenerates `mlflow_runs.csv` with all runs, including the new 20-record/20-trial ones. Useful to attach to your report or open in Excel for a quick sanity check.

📸 **Screenshot opportunity #6:** Open `mlflow_runs.csv` in Excel — filter/sort by `rougeL` descending to show your best-performing hyperparameter combination at a glance.

---

## 7. (Optional) Docker stack for the live demo

If you also want to demo the FastAPI/Prometheus/Grafana stack against the newly-registered model:

```powershell
cd docker
docker compose up --build -d
```

Then in your browser:
- FastAPI Swagger docs: `http://localhost:8000/docs` (or `8002` if you customized ports — check `docker-compose.yml`)
- Grafana: `http://localhost:3000` (`admin` / `admin`)
- Prometheus: `http://localhost:9090`

📸 **Screenshot opportunity #7:** `POST /summarize` in Swagger with a sample paragraph, showing the live generated summary and latency — proves the newly tuned model is actually serving.

📸 **Screenshot opportunity #8:** Grafana dashboard showing request latency/throughput panels after you've hit the API a few times — good "production monitoring" evidence.

When done, shut the stack down to free RAM:
```powershell
docker compose down
```

---

## Summary of what to show in your presentation

| Priority | Screenshot | Why it matters |
|---|---|---|
| ⭐⭐⭐ | MLflow runs table with 20 trials visible | Direct proof of the scale-up |
| ⭐⭐⭐ | Parallel Coordinates Plot (20 trials) | Shows a real hyperparameter search, not a token one |
| ⭐⭐ | Scatter plot (length_penalty vs rougeL) | Supports the "we found a pattern" narrative |
| ⭐⭐ | Model Registry → best version → source run | Ties the registered model back to the 20-trial search |
| ⭐ | Exported CSV sorted by rougeL | Good backup slide / appendix |
| ⭐ | Swagger + Grafana live demo | Only needed if you're doing the full live demo, not just reporting numbers |

---

## Troubleshooting

- **`ModuleNotFoundError: No module named 'src'`** → you forgot `$env:PYTHONPATH="."` in that terminal session, or you're not running the command from the project root.
- **MLflow UI shows nothing new** → check `$env:MLFLOW_TRACKING_URI="http://127.0.0.1:5000"` was set *before* running the scripts, in the same terminal.
- **Very slow first run** → the Hugging Face models and CNN/DailyMail dataset get downloaded and cached the first time; subsequent runs are faster.
- **ZenML pipeline still seems to run only 2 trials** → double check `src/zenml_pipeline.py` has `n_trials=N_TRIALS` in the `train_model(...)` call (search for that exact string) and `N_TRIALS = int(os.getenv("N_TRIALS", "2"))` near the top of the file — these are the two lines the fix added.
