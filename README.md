# Urban Air Pollution Prediction API

FastAPI backend around your trained `RandomForestRegressor` (PM2.5 / PM10,
1h-5h ahead) with Neon (Postgres) as the datastore.

## Why it's shaped this way

Your model needs lag (1-5h), rate-of-change (1-5h), and rolling-mean
(3/6/12/24h) features per station. A single incoming reading can't
produce those alone — the API needs to know what happened in the prior
~24+ hours for that station. So the design is:

1. `POST /readings` stores each raw hourly reading in Neon.
2. `POST /predict/{station_id}` pulls the last 48h of stored readings for
   that station, rebuilds every engineered feature **exactly** the way
   `feature_engineering.py` mirrors your training script, feeds the row
   into the model, and stores + returns the prediction.
3. `POST /ingest-and-predict/{station_id}` does both in one call — the
   endpoint you'll likely wire your ESP32/sensor feed or demo script to.

## Project layout

```
pollution_backend/
├── app/
│   ├── main.py                 FastAPI app + routes
│   ├── config.py                Settings (reads .env)
│   ├── database.py              SQLAlchemy engine/session
│   ├── models.py                ORM tables: sensor_readings, predictions
│   ├── schemas.py                Pydantic request/response models
│   ├── feature_engineering.py   Mirrors your training script's feature code
│   └── ml_service.py             Loads the .pkl artifacts, runs predict()
├── models/                       <- put your 3 .pkl files here
├── seed_from_csv.py              Optional: backfill history from the training CSV
├── requirements.txt
└── .env.example
```

## Setup

1. **Get a Neon database**: console.neon.tech → new project → copy the
   pooled connection string.

2. **Install deps**
   ```bash
   python -m venv venv
   source venv/bin/activate        # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Configure env**
   ```bash
   cp .env.example .env
   # edit .env: paste your Neon DATABASE_URL
   ```

4. **Drop in your trained model artifacts** — copy the three files your
   training script already saves into `models/`:
   - `pollution_model.pkl`
   - `model_features.pkl`
   - `target_columns.pkl`

5. **Run it**
   ```bash
   uvicorn app.main:app --reload
   ```
   Swagger UI: http://127.0.0.1:8000/docs — tables (`sensor_readings`,
   `predictions`) are auto-created in Neon on first startup.

## Demo-ready: seed history from your CSV

`/predict` needs 25+ prior hourly rows per station before it can compute
rolling/lag features. For a hackathon demo, backfill straight from the
same CSV you trained on instead of POSTing readings by hand:

```bash
python seed_from_csv.py /path/to/UrbanAirPollutionDataset.csv --hours-per-station 72
```

Then:
```bash
curl -X POST http://127.0.0.1:8000/predict/1
```

## Typical flow for live/new data

```bash
# 1. push a new hourly reading and get a forecast back in one call
curl -X POST http://127.0.0.1:8000/ingest-and-predict/1 \
  -H "Content-Type: application/json" \
  -d '{
        "station_id": 1,
        "timestamp": "2026-09-06T14:00:00Z",
        "pm25": 58.3, "pm10": 91.0,
        "no2": 22.1, "so2": 9.0, "co": 0.95, "o3": 33.0,
        "temp_c": 30.1, "humidity_pct": 64.0,
        "wind_speed_mps": 2.8, "wind_direction_deg": 195.0,
        "pressure_hpa": 1007.2, "rain_mm": 0.0
      }'

# 2. see prediction history for a station
curl http://127.0.0.1:8000/predictions/1

# 3. see raw reading history for a station
curl http://127.0.0.1:8000/readings/1?hours=24
```

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | liveness check |
| POST | `/readings` | store one raw reading |
| GET | `/readings/{station_id}` | recent readings for a station |
| POST | `/predict/{station_id}` | predict from readings already in DB |
| POST | `/ingest-and-predict/{station_id}` | store + predict in one call |
| GET | `/predictions/{station_id}` | prediction history for a station |

## Notes / things to tighten post-hackathon

- `Base.metadata.create_all` is used instead of Alembic migrations — fine
  for a hackathon, not for prod schema changes.
- `station_id` is assumed to be an int 1-5 matching the `station_1..5`
  one-hot columns from training. If your real Station_ID values are
  strings, adjust `schemas.py` and the `station_*` matching line in
  `feature_engineering.py`.
- CORS is wide open (`allow_origins=["*"]`) for demo convenience — scope
  it down before shipping anywhere real.
- No auth on any endpoint — add an API key dependency if this needs to
  be internet-facing during the hackathon.
# pollution
