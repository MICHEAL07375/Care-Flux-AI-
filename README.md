# 🏥 HospOpt AI — Predict → Optimize → Simulate → Recommend

Hospital resource intelligence: forecasts bed / ICU / ER / OR demand for 1–7 days, then solves an
integer program that turns the forecast into concrete actions under real operating constraints, and lets
administrators stress-test the plan with a what-if simulator.

## Quick start

```bash
# 1) Backend  (Python 3.10+)
cd backend
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
pytest -q                                                 # core + API tests
uvicorn app.main:app --reload --port 8000                 # API docs: http://localhost:8000/docs

# 2) Frontend (Node 18+) - second terminal
cd frontend
npm install
npm run dev                                               # http://localhost:5173  (proxies /api -> :8000)
```

Single-server demo: `npm run build` in `frontend/`, then just run uvicorn — FastAPI serves `frontend/dist` at `/`.

`xgboost` and `pulp` are preferred but optional: if either is missing or fails to run, the backend falls back
to scikit-learn gradient boosting / SciPy HiGHS automatically (same model, same results). The header badges in
the UI show which engine and solver are active.

## Architecture

```
CSV / synthetic data ──► SQLite ──► DemandForecaster (XGBoost) ──► scenario multipliers
                                                                        │
current resources (editable, persisted) ───────────────────────────────►│
                                                                        ▼
                                          per-day integer program (PuLP/CBC, SciPy fallback)
                                                                        ▼
                         actions + reasons + constraint check + before/after ──► React dashboard
```

| File | Role |
|---|---|
| `app/data_gen.py` | Synthetic 180-day dataset (weekly + seasonal pattern + recent demand ramp) |
| `app/forecast.py` | Per-series gradient boosting on deviation from the 7-day mean, recursive multi-step, rolling-origin backtests, 80% band |
| `app/optimizer.py` | The ILP: variables, constraints, costs, human-readable actions, post-solve verification |
| `app/service.py` | Orchestration: scenarios, alerts, summary, impact, CSV ingestion, persistence |
| `app/main.py` | FastAPI endpoints + validation |
| `tests/` | 10 core tests (forecast quality, constraint satisfaction, solver agreement, JSON safety, CSV) + API tests |

## The optimization model (per forecast day)

Levers (integer ≥ 0): `stepdown`, `overflow` ICU beds, regional `transfer`, `defer_adm` (elective admissions),
`defer_surg` (non-urgent surgeries), `nurse_callin`, `doctor_callin`. Slack `un_*` (continuous, heavily penalised)
makes the model always feasible and reports exactly what cannot be covered → shown as **ESCALATE**.

Constraints: ICU patients ≤ ICU beds + overflow · ward patients (including step-down arrivals) ≤ beds − 10 %
emergency reserve · OR hours ≤ hours − 10 % reserve · nurses/doctors required ≤ on duty + call-ins. Staffing couples
beds: moving a patient ICU→ward changes the nurse ratio (1:1.5 → 1:5), which the solver weighs automatically.

Objective: minimise weighted lever cost (`config.COSTS`: step-down < overflow < nurse call-in < defer < transfer ≪ unmet demand).

## API

| Method & path | Purpose |
|---|---|
| `GET /api/health`, `/api/overview`, `/api/history?days=14` | Status, current KPIs, recent series |
| `GET/PUT /api/resources` | Read / partially update capacities, ratios, policies (persisted) |
| `POST /api/forecast` · `/api/optimize` · `/api/simulate` | Body: `{horizon, scenario, resources?}`; simulate returns baseline vs scenario + comparison |
| `GET /api/model/metrics`, `POST /api/model/retrain` | Backtest MAE / MAPE vs seasonal-naive |
| `POST /api/data/upload`, `GET /api/data/export`, `POST /api/data/reset` | Bring your own CSV (needs `date, beds_occupied, icu_occupied, emergency_cases, or_hours_used`, ≥ 60 days) |

Scenario fields: `emergency_surge_pct`, `flu_season_pct`, `staff_absence_pct`, `beds_out_of_service`, `icu_beds_out_of_service`.

## Assumptions to state honestly in the demo

* Data is synthetic. Accuracy numbers are backtests on that data, not clinical validation.
* Surge coupling (+1 % emergencies → ICU ×0.5, beds ×0.25, OR ×0.3) and staffing ratios are editable assumptions
  (`app/config.py`, `app/models.py`).
* Each day is optimised independently (no patient flow carry-over between days) — a clear "future work" item.

## 3-minute demo script

1. Dashboard: current utilisation + alerts ("ICU critical in N days").
2. Forecast chart: ICU tab, 80 % band vs capacity line; mention backtest vs naive.
3. Optimized plan: click the peak day → actions with reasons + constraint check.
4. Press **Emergency +20 %** → heatmap turns red, plan adds step-down / overflow / call-ins, impact table shows shortfall → 0.
5. Drop nurses in *Current resources* → Apply → watch call-ins / ESCALATE appear.
6. Upload your own CSV to show it is not hard-coded.
