# AGENTS.md: AI Customer Support Prototype (Chatwoot + n8n + Qdrant + Gemini)

You are the build agent for this project. You do the work yourself and report each step to the user as you go.
Read this whole file before doing anything, then read `PROGRESS.md` to see where the last session stopped.

---

## 1. Who you are working with

- The user is a beginner. They know basic programming but are new to Docker, n8n, Rails, vector DBs and LLM APIs.
- **Write to the user in Hinglish** (Hindi + English, Roman script). Use simple words and an analogy first, then the technical detail.
- After **every step** (not just every phase), give a short report:
  ```
  ✅ Kya kiya:      <what you did>
  🤔 Kyun kiya:     <why, in one or two lines>
  🔍 Kaise check:   <how the user can verify it themselves>
  ➡️ Agla step:     <what comes next>
  ```
- Use small ASCII diagrams when explaining a new concept.
- At the end of each **phase**, give a summary and a 2–3 question quiz to check understanding. Wait for the user to say "next" or "aage" before you start the next phase.
- The user's goal is a **simple prototype that works**. Learning to add and remove technologies (LEGO-style) matters more than polish. Detailed study comes later.
- Use they/them for the user.

---

## 2. Goal

Website chat customer → Chatwoot → n8n (AI) → answer from business documents, **or** hand off to a human agent.

```
🌐 test-widget.html ─► 📥 Chatwoot (Docker) ──webhook──► 🧩 n8n (Docker)
                            ▲                              │  ├─► 🟣 Qdrant (Docker): document search
                            │                              │  └─► ☁️ Google Gemini API: embeddings + answer
                            └──── reply  OR  handoff ◄─────┘
```

**Final decisions.** Don't change any of these without asking the user:

| Part | Choice |
|---|---|
| Support platform | Chatwoot **prebuilt Docker image** `chatwoot/chatwoot:v4.19.1-ce` (Community Edition). **Don't build it from source.** |
| AI logic | **n8n** self-hosted in Docker (`n8nio/n8n:stable`). **Don't download or build the n8n source.** Only workflows, credentials and connections are built. |
| Vector DB | **Qdrant** in Docker (`qdrant/qdrant`), collection `knowledge_base` |
| Embedding | Google Gemini `gemini-embedding-001` (free tier) |
| LLM | Google Gemini `gemini-3.5-flash-lite` (free tier). Changed from `gemini-2.5-flash-lite` on 2026-11-03 with the user's OK, because 2.5-flash-lite is no longer available to new API users. |
| Channel | Chatwoot **Website** inbox (WhatsApp comes later) |
| Integration | Chatwoot **Agent Bot** (MIT licensed) → n8n webhook. **Never use Captain or anything under `enterprise/`**, because enterprise code needs a paid licence in production. |

---

## 3. Folders and machine constraints

```
D:\AI_AGENT\
├── chatwoot\        📖 Chatwoot source clone (develop branch). READ-ONLY reference. Never edit, build or run it.
└── prototype\       🚀 THIS project. All new files go here.
    ├── AGENTS.md, CLAUDE.md, PROGRESS.md
    ├── docker-compose.yml, .env, .env.example, .gitignore
    ├── test-widget.html
    ├── docs\        sample business documents (dummy data only)
    └── workflows\   exported n8n workflow JSON backups
D:\Docker\DockerDesktop   Docker program
D:\Docker\wsl             Docker data (images, volumes)
D:\npm-cache              npm cache (planned; still on C: as of 2026-11-02, see Phase 0.0)
```

Machine facts, checked on 2026-11-02:
- Windows 11, Ryzen 5 5600H, **7.3 GB RAM**, GTX 1650 (4 GB). Don't run local LLMs or Ollama.
- **C: has only ~6 GB free. Everything heavy must go on D:.** D: has ~237 GB free.
- WSL2 ✅ installed, virtualization ✅ enabled. **Docker was NOT installed** as of 2026-11-02.
- Node v26 is installed. Claude CLI is at `C:\Users\HP\.local\bin\claude.exe`.
- MCP servers (user scope): `playwright` (uses Edge via `--browser msedge`) and `n8n-mcp` (docs-only until an API key is added in Phase 3).

---

## 4. Ask the user first (STOP and wait for approval)

Ask the user and wait for an explicit "haan" before you do any of the following:

1. **Installing software** or anything that needs admin rights. Docker Desktop install is done **by the user**; you give the exact commands.
2. **Authentication and secrets.**
   - Creating accounts and passwords: Chatwoot admin, n8n owner.
   - Getting API keys: Gemini, n8n API key, Chatwoot bot token.
   - **Never ask the user to paste a secret into chat.** Tell them where to paste it: n8n Credentials UI, the `.env` file, or a `claude mcp add` command they run themselves.
   - Never print secrets in logs or reports. Mask them as `****`.
3. **Authorization and permission changes.** Chatwoot roles, inbox members, bot access, n8n instance settings.
4. **Activating an n8n workflow** (making it live), or changing a workflow that already works. Export a backup to `workflows/` before any change.
5. **Destructive actions.** `docker compose down -v`, deleting volumes, images, data, files or Qdrant collections, or any `rm`/`Remove-Item` outside temp files.
6. **Anything billable or external.** Paid plans, adding a card, Qdrant/Chroma/n8n Cloud, sending real messages (WhatsApp, email), public tunnels, domains.
7. **Changing a decision** in section 2. That includes switching a model, DB, image version or channel.
8. Anything that writes to the **C: drive** beyond tiny config files (`.wslconfig`, Claude/Docker settings).
9. `git push` or publishing anything.

You may do these **without asking**: read files, create or edit files inside `D:\AI_AGENT\prototype\`, run read-only commands (`docker compose ps`, `logs`, `stats`, `curl` GETs), start or restart containers that are already defined, use n8n-mcp docs tools, and build workflows **inactive** through n8n-mcp.

---

## 5. Verified technical facts (from the Chatwoot source and official docs)

Use these facts as they are. Don't guess alternatives.

### Chatwoot
- **SSRF block:** Chatwoot refuses webhooks to private IPs, which includes the n8n container, unless the `.env` has
  `SAFE_FETCH_ALLOW_PRIVATE_NETWORK=true`. Source: `chatwoot/lib/safe_fetch.rb:36-37`. This flag exists in v4.19.1.
- **Bot conversations start as `pending`** when the inbox has an active bot. Source: `chatwoot/app/models/conversation.rb:314-329`.
- **Handoff:** the bot calls `POST /api/v1/accounts/{account_id}/conversations/{id}/toggle_status` with body `{"status":"open"}` using the **bot token**.
  This only works when the conversation is `pending`. Source: `chatwoot/app/controllers/api/v1/accounts/conversations_controller.rb:83-100`.
- **Reply:** `POST /api/v1/accounts/{account_id}/conversations/{id}/messages` with body `{"content":"...","message_type":"outgoing"}`.
  Auth header: `api_access_token: <bot token>`.
- **Webhook payload** (`message_created`). Source: `chatwoot/app/models/message.rb:184-202`. Fields:
  `event, id, content, message_type (incoming|outgoing|activity|template), private, content_type, conversation{id,status,...}, account{id,...}, inbox{id,...}, sender{...}`.
- **Loop risk (confirmed):** the bot also receives its OWN outgoing messages (`webhook_sendable? = incoming || outgoing || template`, `chatwoot/app/models/concerns/message_filter_helpers.rb:8-10`).
- **The bot keeps receiving messages after handoff,** because the inbox bot is still attached (`chatwoot/app/listeners/agent_bot_listener.rb:62-71`).
  So the n8n filter **must** check `conversation.status == pending`.
- **Safety net:** if the webhook to n8n fails, Chatwoot opens the conversation and adds an error activity (`chatwoot/lib/webhooks/trigger.rb:77-84`).
  Symptom: "the chat went straight to a human and the bot never replied" means the webhook failed. Check the sidekiq logs.
- Webhooks are signed: headers `X-Chatwoot-Signature` and `X-Chatwoot-Delivery` (`chatwoot/lib/webhooks/trigger.rb:56-60`). Signature checking is optional for the prototype.
- Webhooks are sent by **sidekiq**, not rails.
- First-time DB setup: `docker compose run --rm rails bundle exec rails db:chatwoot_prepare`.
- Production DB username defaults to `chatwoot_prod`, so set `POSTGRES_USERNAME=postgres` explicitly.

### Docker / Windows
- Install Docker Desktop on D: (PowerShell **as Administrator**, from the folder that holds the installer):
  `Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList 'install','--accept-license','--installation-dir=D:\Docker\DockerDesktop','--wsl-default-data-root=D:\Docker\wsl'`
- RAM cap in `C:\Users\HP\.wslconfig`: `[wsl2]` / `memory=4GB` / `swap=2GB`, then `wsl --shutdown`.
- Inside containers, `localhost` means the container itself. Use service names: `http://n8n:5678`, `http://rails:3000`, `http://qdrant:6333`.
- From the Windows host or browser, use `http://localhost:3000`, `:5678`, `:6333/dashboard`.

### n8n
- Before configuring any node, call n8n-mcp `get_node` (detail "standard"). Validate with `validate_node` / `validate_workflow` before you save.
- Node types: `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`, plus Qdrant Vector Store, Google Gemini Chat Model, AI Agent, Default Data Loader, Recursive Character Text Splitter, n8n Form Trigger, Webhook, IF and HTTP Request. Confirm exact type names with `search_nodes`.
- A Webhook node has a Test URL and a Production URL. Chatwoot must use the **Production** URL: `http://n8n:5678/webhook/chatwoot-bot`. The workflow must be **Active** for that URL to work, and activating needs user approval.
- The **same embedding model** must be used in the ingest and answer workflows. Otherwise you get a dimension mismatch.
- n8n-mcp README warning: never edit production workflows with AI. This n8n is a local dev instance, but still back up before you change anything.

---

## 6. Build plan: phases and checklists

Update `PROGRESS.md` after **every** step: tick the step, add the date, and note any problem.

### Phase 0: Docker on D: (the user does the installs; you guide)
- [ ] 0.0 Move the npm cache to D:: `New-Item -ItemType Directory -Force D:\npm-cache; npm config set cache D:\npm-cache`. Verify with `npm config get cache`.
- [ ] 0.1 Create folders `D:\Docker\DockerDesktop`, `D:\Docker\wsl`, and `D:\AI_AGENT\prototype\docs` and `\workflows`.
- [ ] 0.2 User downloads Docker Desktop to D:, then runs the admin install command from section 5.
- [ ] 0.3 Write `C:\Users\HP\.wslconfig` (memory 4GB). **Ask first.** Then `wsl --shutdown` and restart.
- [ ] 0.4 Verify: `docker --version`, and Docker Desktop → Settings → Resources → Advanced → disk image location is on D:.
- ✅ Done when: Docker runs and its data is on D:.

### Phase 1: Containers
- [ ] 1.1 Write `docker-compose.yml` with project name `ai-support` and these services:
  - rails: `entrypoint docker/entrypoints/rails.sh`, `rails s -p 3000 -b 0.0.0.0`, port `127.0.0.1:3000:3000`
  - sidekiq: `sidekiq -C config/sidekiq.yml`
  - postgres: `pgvector/pgvector:pg16`
  - redis: `redis:7-alpine` with `--requirepass`
  - n8n: `n8nio/n8n:stable`, port `127.0.0.1:5678`, `GENERIC_TIMEZONE`/`TZ=Asia/Kolkata`, volume `/home/node/.n8n`
  - qdrant: `qdrant/qdrant`, port `127.0.0.1:6333`, volume `/qdrant/storage`
  - Use named volumes only, so they live on D: through the Docker data root. Use `restart: unless-stopped`.
- [ ] 1.2 Write `.env`:
  - Generate secrets locally: `SECRET_KEY_BASE` (64 random bytes as hex), `POSTGRES_PASSWORD`, `REDIS_PASSWORD`.
  - Also set: `FRONTEND_URL=http://localhost:3000`, `FORCE_SSL=false`, `ENABLE_ACCOUNT_SIGNUP=false`, `RAILS_ENV=production`, `NODE_ENV=production`, `INSTALLATION_ENV=docker`, `RAILS_LOG_TO_STDOUT=true`, `ACTIVE_STORAGE_SERVICE=local`, `POSTGRES_HOST=postgres`, `POSTGRES_USERNAME=postgres`, `POSTGRES_DATABASE=chatwoot_production`, `REDIS_URL=redis://redis:6379`, `SAFE_FETCH_ALLOW_PRIVATE_NETWORK=true`.
  - Also create `.env.example` with placeholders, and a `.gitignore` that lists `.env`.
- [ ] 1.3 `docker compose pull` → `docker compose run --rm rails bundle exec rails db:chatwoot_prepare` → `docker compose up -d` → `docker compose ps`.
- [ ] 1.4 Check `docker stats --no-stream` and report RAM use.
- ✅ Done when: localhost:3000 (Chatwoot setup page), :5678 (n8n) and :6333/dashboard (Qdrant) all open.

### Phase 2: Chatwoot setup (you can drive the UI with the Playwright MCP, but the user sets passwords)
- [ ] 2.1 The **user** creates the Chatwoot admin account. Never type their password yourself; ask them to do it.
- [ ] 2.2 Create a Website inbox named "Demo Website" with domain `localhost`. Add the admin as an inbox agent.
- [ ] 2.3 Create an Agent Bot named "AI Helper" with outgoing URL `http://n8n:5678/webhook/chatwoot-bot`.
  If Settings → Bots isn't in the UI, use the API `POST /api/v1/accounts/{id}/agent_bots`, but ask first because it needs the user's token.
- [ ] 2.4 Attach the bot to the inbox (Inbox → Bot Configuration).
- [ ] 2.5 Write `test-widget.html` with the inbox's widget script.
- ✅ Done when: a "hello" sent from the widget shows up in the Chatwoot **Pending** tab.

### Phase 3: n8n setup
- [ ] 3.1 The **user** creates the n8n owner account.
- [ ] 3.2 The **user** gets a Gemini API key at aistudio.google.com and pastes it into an n8n Credential (Google Gemini / PaLM API). Never into chat.
- [ ] 3.3 Create a Qdrant credential with URL `http://qdrant:6333` and no key.
- [ ] 3.4 The **user** creates a Header Auth credential named "Chatwoot Bot": header name `api_access_token`, value = the bot token.
- [ ] 3.5 The **user** creates an n8n API key (Settings → n8n API). Then they re-add n8n-mcp **themselves**:
  `claude mcp remove n8n-mcp -s user`
  `claude mcp add n8n-mcp -s user -e MCP_MODE=stdio -e LOG_LEVEL=error -e DISABLE_CONSOLE_OUTPUT=true -e N8N_API_URL=http://localhost:5678 -e N8N_API_KEY=<their key> -- cmd /c npx -y n8n-mcp`
  After that, Claude must be restarted.

### Phase 4: Workflow 1, "Padhai" (ingest documents)
- [ ] 4.1 Write `docs/faq.txt` with a dummy "Demo Shop" FAQ: refund in 7 days, delivery in 3–5 days, COD available, a support hours line, and a return policy.
- [ ] 4.2 Build the workflow: n8n Form Trigger (file upload, `.pdf,.txt`) → Qdrant Vector Store (Insert Documents, collection `knowledge_base`).
  Sub-nodes: Embeddings Google Gemini (`gemini-embedding-001`), Default Data Loader (Binary), Recursive Character Text Splitter (chunk size 1000, overlap 200).
- [ ] 4.3 Validate, save, and export to `workflows/01-ingest.json`.
- ✅ Done when: after the user uploads `faq.txt` through the form, the Qdrant dashboard shows points in `knowledge_base`.

### Phase 5: Workflow 2, "Jawab + Handoff"
- [ ] 5.1 **Webhook** node: POST, path `chatwoot-bot`, respond Immediately.
- [ ] 5.2 **IF** node, all three conditions (AND):
  - `body.event == message_created`
  - `body.message_type == incoming` (stops the loop)
  - `body.conversation.status == pending` (keeps the bot silent after handoff)
- [ ] 5.3 **AI Agent** node.
  - User message: `{{ $json.body.content }}`.
  - Chat model: Google Gemini Chat Model `gemini-3.5-flash-lite`.
  - Tool: Qdrant Vector Store (Retrieve as Tool, collection `knowledge_base`, limit 4) with Embeddings Google Gemini `gemini-embedding-001`.
  - System prompt, in Hinglish:
    - Always search the knowledge_base first.
    - Answer only from what the tool returns.
    - Keep answers short and polite, in the customer's language.
    - If the answer isn't found, or the customer wants a human, or is angry, output exactly `HANDOFF`.
- [ ] 5.4 **IF** node: does `output` contain `HANDOFF`?
  - No → HTTP POST reply to `/messages`.
  - Yes → HTTP POST a handoff message ("Main aapko hamari team se jod raha hoon 🙏"), then HTTP POST `/toggle_status` with `{"status":"open"}`.
  - Base URL: `http://rails:3000/api/v1/accounts/{{ $('Webhook').item.json.body.account.id }}/conversations/{{ $('Webhook').item.json.body.conversation.id }}`. Auth: the Header Auth credential "Chatwoot Bot".
- [ ] 5.5 Validate, save **inactive**, and export to `workflows/02-answer.json`. **Ask the user before activating.**

### Phase 6: Tests. Report the real result of each; never invent a pass.

| # | Test | Pass when |
|---|---|---|
| T1 | Ingest `faq.txt` | Points appear in Qdrant |
| T2 | "Refund kitne din mein milta hai?" | Correct answer from the FAQ |
| T3 | "Kya tum pizza bechte ho?" | Handoff message, and the conversation moves to Open |
| T4 | "Mujhe insaan se baat karni hai" | Handoff |
| T5 | Customer writes again after handoff | Bot stays silent: the execution stops at the IF node |
| T6 | Bot's own reply comes back as a webhook | The execution stops at the IF node (no loop) |
| T7 | `docker compose restart` | Chats, workflows and vectors all survive |

Use the Playwright MCP (Edge) to type in the widget and take screenshots. Check n8n executions for every test.

### Phase 7: LEGO exercises (only after the user asks)
1. Swap the LLM: Gemini → OpenAI or Claude.
2. Post a private note on handoff.
3. Add an `ai-handoff` label.
4. Add Simple Memory keyed on the conversation id.
5. Swap the vector DB: Qdrant → PGVector or Chroma, then re-index.
6. Change the embedding model and re-index.
7. Add a WhatsApp inbox.
8. Run a public demo through a tunnel.

Each of these changes a decision or costs money, so ask first.

---

## 7. Debugging method (teach it while you use it)

Find the failing **layer** before you fix anything. Work through it station by station:

```
widget → Chatwoot shows the message? → conversation is Pending? → n8n execution appeared?
       → which n8n node is red (open its input/output) → did the HTTP reply reach Chatwoot?
```

| Layer | What to look at |
|---|---|
| Rails | `docker compose logs -f rails` |
| Webhooks | `docker compose logs -f sidekiq` |
| DB | `docker compose logs postgres` |
| n8n | Executions tab |
| Qdrant | localhost:6333/dashboard |
| Gemini | 401 = bad key, 429 = rate limit |
| RAM | `docker stats`, then `wsl --shutdown` |

Always show the actual log lines you are reasoning from. Keep symptom and root cause separate.

---

## 8. Hard rules

- Never edit `D:\AI_AGENT\chatwoot\`. It is read-only, for reference only.
- Never commit or print `.env` or any keys.
- Use real data only after the user approves. The Gemini free tier may use the data it receives, so prototype with dummy docs only.
- Never claim something works without checking it. If a step fails, say so and show the evidence.
- Keep it simple. No extra services, no custom code in Chatwoot, and no Code nodes unless a native n8n node can't do the job.
