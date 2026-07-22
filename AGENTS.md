# Agent Instructions

This project uses **bd** (beads) for issue tracking. Run `bd onboard` to get started.

## Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --status in_progress  # Claim work
bd close <id>         # Complete work
bd sync               # Sync with git
```

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd sync
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds

## Cursor Cloud specific instructions

This is a document ingestion pipeline: FastAPI `api` (control plane) + Temporal `worker`, backed by SQLite (canonical state), MinIO (artifacts), Temporal (orchestration), Marqo (search), plus a Node `lang-detect` service and a React/Vite `ui`. Standard run/build/test commands live in `README.md`; only non-obvious cloud caveats are below.

### What can and cannot run here
- No Docker and no GPU in the cloud VM, so the `docker compose` stack does NOT run: the `marqo` image is a GPU-only local build, and OCR/translation/chunking call external GPU LLM endpoints (Chandra `:8010`, Gemma `:8020`, Qwen `:8005`) that are not provided. Run services natively in dev mode instead (see below).
- The reachable end-to-end flow is the control plane: upload/register a document, it lands in SQLite + MinIO and a Temporal workflow starts. Downstream OCR/translation/chunking/search stages fail/stall without the external LLM endpoints and Marqo — that is expected here, not a setup bug.

### Backend (Python)
- Use the `.venv` virtualenv (Python 3.12). Run tests with `.venv/bin/pytest`, compile-check with `.venv/bin/python -m py_compile pipeline/*.py`. There is no configured linter.
- The API endpoint tests in `tests/test_api.py` exercise the real FastAPI lifespan, which connects to a live Temporal AND a live MinIO. Without them, those ~21 tests error (the other ~78 pass standalone). To get all 99 passing you must have Temporal running on `localhost:7233` and MinIO running on `localhost:9000` with access key `test-key` / secret `test-secret` (these are hardcoded in `tests/test_api.py`; any other MinIO creds cause `InvalidAccessKeyId`).

### Running services natively (dev)
- Temporal: `temporal server start-dev --port 7233 --ui-port 8080` (CLI installed at `~/.temporalio/bin/temporal`).
- MinIO: run the `minio` binary as `MINIO_ROOT_USER=test-key MINIO_ROOT_PASSWORD=test-secret minio server <datadir> --address :9000 --console-address :9001` so it matches the test/app creds.
- `lang-detect`: default port is 3000, which collides with the UI. Run it on `PORT=3001 npm start` (the app's default `LANG_DETECT_URL` is `http://lang-detect:3001`).
- `api`: `uvicorn pipeline.api:app --host 0.0.0.0 --port 8001`. Required env: `MINIO_ACCESS_KEY`/`MINIO_SECRET_KEY` (else it refuses to start), plus `MINIO_ENDPOINT=localhost:9000`, `TEMPORAL_HOST=localhost:7233`, `DOCUMENT_DB_PATH` (a writable path, e.g. `/tmp/pipeline-data/documents.db`), `LANG_DETECT_URL=http://localhost:3001`, `AUTH_DISABLED=true`. The lifespan auto-creates the MinIO bucket.
- `worker`: `python -m pipeline.worker`. It refuses to start unless `TRANSLATION_VLLM_BASE_URL` is set (set a placeholder like `http://localhost:8020/v1` even though no real endpoint exists).
- `ui`: `npm run dev` serves on port 3000 and proxies `/api`→api and `/marqo`→marqo. For local dev set `VITE_API_PROXY_TARGET=http://localhost:8001` (the default `http://api:8001` only resolves inside compose).
- Auth is off by default (`AUTH_DISABLED=true`); Keycloak is not needed for normal dev.

