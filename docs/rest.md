# REST interface

`snoextract-server` exposes a JSON HTTP API. Data loads once at startup, so per-note cost is just inference.show
## Start the server

REST is **off by default** — `--http-listen` opts in:

```bash
# Linux / macOS
./snoextract-server --http-listen 127.0.0.1:50052

# Windows (cmd)
snoextract-server.exe --http-listen 127.0.0.1:50052
```

Loopback-only — no auth, no TLS. For cross-machine access, put a reverse proxy (nginx, caddy, envoy) in front and terminate TLS there.

## Health check

```bash
curl -s http://127.0.0.1:50052/v1/health
# => {"status":"ok"}
```

## Extract entities

```bash
# Linux / macOS
curl -s -X POST http://127.0.0.1:50052/v1/extract \
  -H 'content-type: application/json' \
  -d '{"text":"Patient with chest pain and hypertension."}'
```

```cmd
:: Windows (cmd)
curl -s -X POST http://127.0.0.1:50052/v1/extract -H "content-type: application/json" -d "{\"text\":\"chest pain\"}"
```

Output (truncated):

```json
{
  "entities": [
    { "text": "chest pain", "start": 13, "end": 23, "cui": "29857009",
      "name": "Chest pain (finding)", "semantic_type": "finding", "similarity": 1.0,
      "context": { "certainty": "affirmed", "is_negated": false, "is_uncertain": false, "is_historical": false, ... } },
    { "text": "hypertension", "start": 28, "end": 40, "cui": "38341003",
      "name": "Hypertensive disorder, systemic arterial (disorder)", "semantic_type": "disorder", "similarity": 1.0,
      "context": { "certainty": "affirmed", "is_negated": false, "is_uncertain": false, "is_historical": false, ... } }
  ],
  "sections": [], "heat_map": [], "suppressed": []
}
```

A minimal call is just `{"text":"..."}` — missing fields fall back to defaults. Optional fields: `min_similarity` (per-call linker threshold) and `doc_id` (echoed back). The response shape mirrors the `ExtractResponse` message in `proto/snoextract/v1/snoextract.proto`.
