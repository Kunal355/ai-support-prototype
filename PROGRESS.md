# PROGRESS.md

The build agent updates this file after every step. Each entry gets: the date, what was done, how it was verified, and any problems.

## Current status
- Phase: **7 in progress**. Exercises 2+3 done (private note + `ai-handoff` label). The user picks the next exercise.
- Next step: none required. Wait for the user (Phase 7 ideas, or a deeper study).
- After any n8n restart, the first webhook can take ~2 s while n8n registers it; Chatwoot's webhook timeout is 5 s.
- IDs: account_id=2, inbox_id=1 ("Demo Website"), agent_bot_id=1 ("AI Helper").
- Widget test page: run `python -m http.server 8080 --bind 127.0.0.1` in this folder, open http://localhost:8080/test-widget.html (Playwright blocks `file://`).
- Start everything after a reboot: open Docker Desktop, then `docker compose up -d` in this folder.

## Done before this file existed (planning session, 2026-11-02)
- [x] Cloned Chatwoot to `D:\AI_AGENT\chatwoot` (read-only reference).
- [x] Decided the stack: Chatwoot v4.19.1-ce image, n8n, Qdrant, Gemini free tier, website widget, Agent Bot handoff.
- [x] npm cache moved to `D:\npm-cache` (Phase 0.0, 2026-11-02).
- [x] Added MCPs (user scope): `playwright` (Edge) and `n8n-mcp` (docs-only). Both showed ✔ Connected on 2026-11-02.
- [x] Created `AGENTS.md`, `CLAUDE.md` and this file.

## Log
<!-- Add new entries below, newest at the bottom. Format:
### YYYY-MM-DD: Phase X.Y <title>
- Did: ...
- Verified: ...
- Issues: ...
-->

### 2026-11-02: Phase 0 Docker on D: (done)
- [x] 0.0 Did: `npm config set cache D:\npm-cache`. Verified: `npm config get cache` → `D:\npm-cache`.
- [x] 0.1 Did: created `D:\Docker\DockerDesktop`, `D:\Docker\wsl`, `D:\Docker\installer`, `prototype\docs`, `prototype\workflows`.
- [x] 0.2 Did: the user asked the agent to install Docker itself. Downloaded the installer to `D:\Docker\installer` (591 MB, Authenticode signature Valid, signer Docker Inc). Ran it elevated with `--installation-dir=D:\Docker\DockerDesktop --wsl-default-data-root=D:\Docker\wsl`. Exit code 0.
- [x] 0.3 Did: wrote `C:\Users\HP\.wslconfig` (memory=4GB, swap=2GB), then `wsl --shutdown`.
- [x] 0.4 Verified: Docker 29.9.2, Compose v5.6.1, engine MemTotal ≈ 4 GB, `hello-world` ran. Data disks are `D:\Docker\wsl\disk\docker_data.vhdx` and `D:\Docker\wsl\main\ext4.vhdx`.
- Issues: `docker` is not on PATH in terminals opened before the install (open a new terminal). PowerShell 5.1 shows docker's progress output as red errors; harmless.

### 2026-11-02: Phase 1 Containers (done)
- [x] 1.1 Did: wrote `docker-compose.yml` (project `ai-support`: rails, sidekiq, postgres, redis, n8n, qdrant; named volumes only; `restart: unless-stopped`).
- [x] 1.2 Did: wrote `.env` with secrets generated locally by `openssl rand` (never printed), `.env.example` with `change_me` placeholders, `.gitignore` with `.env`. Verified: `docker compose config --quiet` OK.
- [x] 1.3 Did: `docker compose pull`, `docker compose run --rm rails bundle exec rails db:chatwoot_prepare` (exit 0), `docker compose up -d`.
- [x] 1.4 RAM (`docker stats`, just after start): n8n ~480 MB, sidekiq ~430 MB, rails ~400 MB, postgres ~65 MB, qdrant ~60 MB, redis ~9 MB. Total ≈ 1.45 GB of the 4 GB WSL cap.
- Verified: http://localhost:3000 → 302 to onboarding (200), http://localhost:5678 → 200, http://localhost:6333/dashboard → 200 (Qdrant 1.19.0).
- Issues:
  - Docker Desktop had stopped between steps (`dockerDesktopLinuxEngine` pipe missing). Fix: start Docker Desktop again.
  - db:chatwoot_prepare logs `PG::UndefinedTable ... installation_configs` once at boot, before the schema exists. Harmless; the task finished with exit 0.
  - **Qdrant `latest` (v1.20.0) crash-looped with `exec ./entrypoint.sh: exec format error`.** Root cause: its unpacked filesystem had 0-byte files (container export ≈ 2 MB instead of ~270 MB), while the downloaded blobs were fine. Re-pull did not help. `qdrant/qdrant:v1.19.0` works, so compose is **pinned to v1.19.0** (deviation from section 2's untagged `qdrant/qdrant`; revert by changing the tag). The broken `qdrant/qdrant:latest` image is still on disk (~276 MB); removing it needs the user's OK.

### 2026-11-02: Phase 2 Chatwoot setup
- [x] 2.1 The user created the admin account and the account "zaplet" through onboarding (account_id=2).
- [x] 2.2 Did: created Website inbox "Demo Website" (website_url `http://localhost`, inbox_id=1) and added the admin as inbox member. Done with a `rails runner` script (models only, no Chatwoot code changed) instead of clicking through the UI.
- [x] 2.3 Did: created Agent Bot "AI Helper" (agent_bot_id=1), outgoing URL `http://n8n:5678/webhook/chatwoot-bot`. The bot access token exists; it is not printed anywhere.
- [x] 2.4 Did: attached the bot to the inbox (AgentBotInbox status active).
- [x] 2.5 Did: wrote `test-widget.html` (widget script with the inbox website token).
- Verified (Playwright, Edge): widget opened, "hello" sent, conversation 1 created in inbox 1 and assigned to "AI Helper". Sidekiq called `http://n8n:5678/webhook/chatwoot-bot` and got **404 Not Found**, because no n8n workflow exists yet. Chatwoot's safety net then opened the conversation ("Conversation was marked open by system due to an error with the agent bot."). This shows the private-network webhook works (SAFE_FETCH flag OK); a 404, not an SSRF block.
- Not yet verified: "hello lands in **Pending**". Needs a live n8n webhook; re-test after Phase 3/5.
- Issues: Playwright cannot open `file://`, so the page is served with Python on port 8080. Widget console shows one harmless 404 on `toggle_typing` before the conversation existed. Sidekiq logs print the bot's webhook secret in job arguments; don't share those logs.

### 2026-11-02: Prep work while waiting for 3.1
- [x] 4.1 Did: wrote `docs/faq.txt` (dummy Demo Shop FAQ: refund 7 days, return 10 days, delivery 3-5 days, COD up to Rs 5000, support hours, payment methods, cancellation, tracking).
- Did: drafted `workflows/01-ingest.json` and `workflows/02-answer.json` (not imported yet, inactive). Node versions checked against the n8n 2.45.7 container and n8n-mcp `get_node`. Verified: n8n-mcp `validate_workflow` → valid, 0 errors, 0 warnings for both.
- Credentials are not linked yet; they get linked after the user creates them in Phase 3.
- Idea for 3.5: workflows can be imported with `n8n import:workflow` inside the container, so the n8n API key is optional. Ask the user.

### 2026-11-02: Phase 3 n8n setup
- Docker Desktop was stopped again at session start; started it, `docker compose up -d`, all 6 services Up.
- [x] 3.1 The user created the n8n owner account. Verified: `/rest/settings` showSetupOnFirstLoad=false.
- [x] 3.3 Did: imported credential "Qdrant Local" (id `qdrantLocal00001`, type qdrantApi, URL `http://qdrant:6333`, no key) with `n8n import:credentials`. Verified: `n8n export:credentials` lists it.
- [x] 3.2 The user created the Google Gemini(PaLM) Api credential. Verified by the user: "Connection tested successfully" (screenshot).
- [x] 3.4 Verified 2026-11-03: credential "Chatwoot Bot" (httpHeaderAuth, id `2FFHJgFhzZ2i3oBp`) exists. User decided to create the "Chatwoot Bot" Header Auth credential themselves (token from Chatwoot → Settings → Bots → AI Helper → edit, Access Token copy button).
- [x] 3.5 Re-checked later on 2026-11-03: `claude mcp get n8n-mcp` now shows `N8N_API_URL=http://localhost:5678` and `N8N_API_KEY=****`, status ✔ Connected. A Claude restart is needed before the n8n management tools appear. (Earlier the same day it showed no key, so workflow 1 was imported with `n8n import:workflow`.)

### 2026-11-03: Phase 4 Workflow 1 "Padhai"
- [x] 4.1 Done earlier (`docs/faq.txt`).
- [x] 4.2 Did: linked credentials in `workflows/01-ingest.json` (Qdrant Local, Google Gemini(PaLM) Api account), set workflow id `ingestWorkflow01`. Imported with `n8n import:workflow` (inactive). Verified: `n8n list:workflow` → `ingestWorkflow01|01 Padhai - Ingest documents`.
- [x] 4.3 Verified: n8n-mcp `validate_workflow` → valid, 0 errors, 0 warnings. Export = `workflows/01-ingest.json`.
- [x] Done-check (2026-11-03): the user uploaded faq.txt via the form. Verified: `knowledge_base` exists, points_count=2, vector size 3072, Cosine. Chunk 1 = lines 1-17 (about, refund, return, delivery, COD), chunk 2 = lines 16-29 (COD, tracking, cancel, payment, support hours). The COD section shows up in both chunks because of the 200-char overlap. Before upload: Qdrant collections = `[]` (empty).
- Issues: a copy of the workflow JSON (no secrets) stays at `/tmp/01-ingest.json` in the n8n container (root-owned, `rm` not permitted); gone on container recreate.
- 2026-11-03 (later): re-checked. Qdrant collections still `[]`, n8n logs show no execution yet, so the upload hasn't happened. n8n-mcp management tools now load but fail with `SSRF protection: Localhost access is blocked in strict mode`. Likely fix: user re-adds n8n-mcp with `-e WEBHOOK_SECURITY_MODE=moderate` (not done; optional, CLI import works).

### 2026-11-03: Phase 5 Workflow 2 "Jawab + Handoff"
- [x] 5.1-5.4 Built earlier as a draft (Webhook POST `chatwoot-bot` respond immediately → IF event/incoming/pending → AI Agent with Gemini 2.5 flash-lite + Qdrant retrieve-as-tool topK 4 + gemini-embedding-001 → IF output contains HANDOFF → reply, or handoff message + toggle_status open).
- [x] Did: linked credentials (Gemini `TVLoFE2T2FDezT6s`, Qdrant `qdrantLocal00001`, Chatwoot Bot `2FFHJgFhzZ2i3oBp` on all 3 HTTP nodes), set id `answerWorkflow02`. Backup of the uncredentialed draft in the session scratchpad.
- [x] 5.5 Verified: n8n-mcp `validate_workflow` → valid, 0 errors, 0 warnings. Imported with `n8n import:workflow` (inactive). `n8n list:workflow` → `answerWorkflow02|02 Jawab + Handoff`. Export = `workflows/02-answer.json`.
- [x] Activation: the user said "haan". `n8n publish:workflow --id=answerWorkflow02` + `docker compose restart n8n`. Verified: log "1 published workflows", POST /webhook/chatwoot-bot → 200 `Workflow was started`.

### 2026-11-03: Phase 6 Tests
- [x] T1 Ingest: passed in Phase 4 (2 points in `knowledge_base`).
- [x] Phase 2 leftover verified: a new widget conversation (conv 3) stays **pending** with the bot attached, and sidekiq delivers webhooks to n8n in ~130-310 ms.
- [ ] T2 "Refund kitne din mein milta hai?" **FAILED, no reply.**
  - Try 1 (conv 2): sidekiq `Net::ReadTimeout` on the webhook (Chatwoot timeout = 5 s, `lib/webhooks/trigger.rb:118-122`). Chatwoot's safety net opened the conversation. Likely the first webhook after the n8n restart was slow (cold start); later calls took <0.5 s.
  - Try 2 (conv 3): webhook OK. The n8n event log shows execution 17: `n8n.ai.llm.error` on Gemini Chat Model, "The resource you are requesting could not be found".
  - Root cause (direct API test, key not printed): `gemini-2.5-flash-lite` → 404 "no longer available to new users, use models/gemini-3.5-flash-lite". `gemini-3.5-flash-lite` and `gemini-flash-lite-latest` → 200. `gemini-embedding-001` is still listed.
  - Fix: the user approved `gemini-3.5-flash-lite` ("haan, 3.5"). Backup: `workflows/02-answer.backup-2026-11-03-gemini25.json`. Changed only the Gemini Chat Model node, re-imported, `publish:workflow`, restarted n8n.
- [x] T2 (conv 4): reply "Refund, return kiya hua product milne ke 7 din ke andar process kar diya jata hai." Matches the FAQ. Screenshot `test-screens/T2-refund.png`.
- [x] T3 (conv 4): "Kya tum pizza bechte ho?" → "Main aapko hamari team se jod raha hoon 🙏", activity "Conversation was reopened by AI Helper", status open. Screenshot `test-screens/T3-pizza.png`.
- [x] T4 (conv 5): "Mujhe insaan se baat karni hai" → handoff message, status open. n8n exec 45 path: Webhook > IF > Agent > HANDOFF? > Handoff message > Open for human. Screenshot `test-screens/T4-human.png`.
- [x] T5 (conv 4): "Hello? koi hai?" after handoff → no bot reply. n8n exec 43 stopped at "Customer msg + Pending?".
- [x] T6: the bot's own outgoing messages and activity events (execs 46-56) all stopped at "Customer msg + Pending?". No loop.
- [x] T7: `docker compose restart`. After restart: Qdrant points=2, both workflows listed, webhook → 200, Chatwoot 5 conversations / 25 messages kept. New conv 6 "Delivery kitne din mein hoti hai?" → "Metro cities mein delivery 3 se 5 working days aur doosri jagahon par 5 se 7 working days mein hoti hai." Screenshot `test-screens/T7-after-restart.png`.
- Note: n8n-mcp management tools still fail with the SSRF strict-mode error; executions were read from `/home/node/.n8n/n8nEventLog.log` instead.

### 2026-11-03: Phase 7 LEGO, exercises 2+3 (handoff note + label)
- The user chose "Handoff note + label".
- Source check: the bot token may call `conversations/labels` index/create and `messages` create (`chatwoot/app/controllers/concerns/access_token_auth_helper.rb:2-7`). `private` is a message param (`app/builders/messages/message_builder.rb:10`). Labels `create` **replaces** the conversation's labels (`labelable.rb:9-11`); fine because bot chats have no labels at handoff.
- Did: backup `workflows/02-answer.backup-2026-11-03-before-phase7.json`. Added 2 HTTP nodes after "Open for human": "Private note for agent" (POST /messages, private=true, includes the customer's last message) → "Add ai-handoff label" (POST /labels `{"labels":["ai-handoff"]}`). validate_workflow → 0 errors. Created Chatwoot label `ai-handoff` (red, sidebar) with rails runner. Re-imported, published, restarted n8n.
- Verified (conv 7, "Kya aap laptop repair karte ho?"): the customer saw only the handoff message. DB: private note private=true, status open, labels ["ai-handoff"], activity "AI Helper added ai-handoff".
