# DocuAgent — Work Session Reference

> Session date: **2026-07-06** · Use this as the starting context for the next chat.
> Git branch: `performance-improve` · **All 7 phases (0–6) are DONE and committed. Phase 6 (`07fe2b3`) is committed locally but NOT yet pushed — run `git push` when ready (verified safe: no secrets/internal docs in the diff).**

**Commit map (phase → commit):**
| Phase | Commit | |
|-------|--------|--|
| 0 Environment & cleanup | `568ed21` | orphan cleanup, AiAgent PORT fix, `.env.example` files |
| 1 Security hardening | `d16ceee` | mandatory `JWT_SECRET`, 5MB upload limit |
| 2 API routing | `c0c123f` | relative `/api`, Vite proxy, prod SPA serve |
| 3 + 4 Auth + Python fixes | `cdc0e5a` | signup username, reportlab PDF, bold DOCX, JSON hardening, drop PPTX |
| 5 UX polish | `2fd61a4` | phased progress, single-file input, HistoryPage logout, empty-state border |
| 6 Structural cleanup | `07fe2b3` | flatten `Client/Client`, remove artifacts, drop `parseInfo`/tree-sitter |

---

## 1. What this project is

DocuAgent turns source code + instructions into generated documentation (DOCX / PDF).
Three-tier architecture:

```
Client (React/Vite)  ->  Server (Node/Express gateway)  ->  Python services
                                                             ├─ Docbuilder.py  (orchestrates build, renders DOCX/PDF)
                                                             ├─ AiAgent.py     (calls Gemini -> markdown)
                                                             └─ Uml.py         (diagram images)
```

Request flow: `Client → Server.js /generate → Docbuilder.py /build-document → AiAgent.py /generate-doc (markdown) + Uml → DOCX/PDF back`.

---

## 2. The plan we are executing

Source of truth: `DocuAgent_Project_Plan.docx` and `flags.md` (issue registry) in the repo root.
Dependency order: **Phase 0 → 1 → 2 → (3 ‖ 4) → 5 → 6**

| Phase | Title | Status |
|-------|-------|--------|
| 0 | Environment & cleanup | ✅ Done |
| 1 | Security hardening | ✅ Done |
| 2 | API routing (local-dev) | ✅ Done |
| 3 | Auth consistency | ✅ Done |
| 4 | Python service fixes | ✅ Done |
| 5 | UX polish | ✅ Done |
| 6 | Structural cleanup (optional) | ✅ Done (committed, unpushed) |

---

## 3. Our workflow (repeat this per phase)

1. **Read** the relevant files first (never edit blind).
2. **Apply** the phase's implementation steps — extend beyond the literal plan only when the phase's own test criteria require it (and call it out).
3. **Verify behaviorally**, not just syntax:
   - JS: `node --check`, plus isolated harnesses reusing the repo's own deps (`axios`, `form-data`) from `Server/node_modules`.
   - Python: `py -m py_compile`; for runtime behavior, a **throwaway venv** in the scratchpad (local dev deps like reportlab are NOT installed in the system `py`), then clean it up.
4. **Report** what changed, what was verified, and what couldn't be (e.g. full e2e needing live Gemini/Mongo).
5. Provide a **conventional commit message** per phase. (User has been collecting them; nothing committed yet.)

Notes:
- OS is Windows; shell is PowerShell primary, Bash tool available. `python` isn't on PATH — use **`py`**.
- Client was flattened in Phase 6 — it now lives at `Client/` (was `Client/Client/`).

---

## 4. Completed work — file-by-file

### Phase 0 — Environment & cleanup
- **Deleted** 0-byte orphans: `Service/HistoryPage.jsx`, `Service/Server.js`, `Service/UploadPage.jsx`
- `Service/AiAgent.py` — startup `PORT` now `int(os.getenv("PORT", 5001))` (was `os.environ["PORT"]` → KeyError)
- `Server/Server.js` — `mongoose.connect(MONGO_URI)` (dropped deprecated `useNewUrlParser`/`useUnifiedTopology`)
- **Created** `Server/.env.example` — keys: `PORT`, `MONGO_URI`, `JWT_SECRET`, `DOC_BUILDER_URL`, `NODE_ENV`
- **Created** `Service/.env.example` — keys: `GEMINI_API_KEY`, `PORT`, `DOCBUILDER_URL`, `PARENT_AGENT_URL`, `UML_AGENT_URL`, `STATIC_DIR`
- **Created** root `.gitignore` (node_modules, `__pycache__`, `.env`, version artifacts `0.5.1`/`1.26.0`, output dirs)
- `README.md` — env section rewritten to match real variables (two `.env` files)

### Phase 1 — Security hardening (`Server/Server.js`)
- `JWT_SECRET` read from env; if missing → `console.error("FATAL: ...")` + `process.exit(1)`
- multer: `limits: { fileSize: 5 * 1024 * 1024 }`
- Added error-handling middleware → `413` JSON on `LIMIT_FILE_SIZE`
- Verified: no-secret start exits 1; 6MB → 413; 1MB → 200

### Phase 2 — API routing
- `Client/Client/vite.config.js` — proxy target → `http://localhost:3001` (rewrite already strips `/api`)
- Absolute `https://mainserver-kpei.onrender.com` → relative `/api/*` in: `UploadPage.jsx`, `HistoryPage.jsx`, **`LoginPage.jsx`, `SignupPage.jsx`, `LandingPage.jsx`** (extended beyond plan so no request hits prod in dev)
- `Server/Server.js` — when `NODE_ENV==='production'`: serve `Client/Client/dist` static + SPA fallback (`app.get(/.*/ )`, Express-5 safe) + **`/api` prefix strip** (prod has no Vite proxy)

### Phase 3 — Auth consistency (`Client/Client/src/UploadPage.jsx`)
- Added `authUsername` state
- Username `<input>` rendered when `authMode === "signup"`
- `username` added to `POST /api/signup` body
- `authUsername` cleared in `handleLogout`

### Phase 4 — Python service fixes
- `Service/Docbuilder.py`:
  - Removed `docx2pdf` + `pptx` imports; added reportlab platypus imports + `re`/`escape`
  - New helpers: `parse_inline_md()` (shared), `_inline_to_rl()`, `generate_pdf(raw_md, pdf_path)`
  - DOCX body now builds bold/italic **runs** (was `.replace('*','')`)
  - PDF now `generate_pdf(raw_md, pdf_path)` — reportlab, no MS Word
  - **Removed** empty-PPTX generation + pptx paths/response keys
- `Service/AiAgent.py` — `request.get_json(force=True, silent=True)` + `if not isinstance(data, dict): return 400`
- `Client/Client/src/UploadPage.jsx` — removed PPTX from `formatOptions` + unused `FiFilePlus` import
- `Service/requirements.txt` — removed `python-pptx` and `docx2pdf`
- Verified in venv: real PDF (valid `%PDF-`), DOCX bold runs present, JSON guard returns clean 400s

### Phase 5 — UX polish (`Client/src/UploadPage.jsx`, `Client/src/HistoryPage.jsx`) — commit `2fd61a4`
- **Phased progress** — added `generationPhase` state (`"uploading"` → `"processing"`). Shows real upload `%`, then flips to an indeterminate animated bar + "AI is generating…" message once upload hits 100% (previously froze at 100% through the multi-minute AI step). Button label: `Uploading… %` → `Generating…`. Reset in `finally`.
- **Single-file input** — removed `multiple` from `<input>`; labels singular ("Upload Your File", "Drag & Drop your file", "Uploaded File"). Only `files[0]` was ever sent, so behavior unchanged.
- **HistoryPage logout** — added `FiLogOut` import, `handleLogout` (clears token → `/login`), logout button in header (replaced empty `w-10` spacer).
- **Empty-state border** — added `border` base class alongside `border-dashed` (dashed border never rendered without it).
- Verified: `vite build` passes.

### Phase 6 — Structural cleanup — commit `07fe2b3`
- **Flattened** `Client/Client/*` → `Client/*` (all 21 tracked files as 100% renames). Nested `Client/Client/` dir removed. `node_modules`/`dist` relocated too (untracked).
- Deleted junk 85-byte `Client/package-lock.json`; real lockfile now at `Client/package-lock.json`.
- `git rm` version artifacts `0.5.1`, `1.26.0` (already gitignored).
- `Server/Server.js` prod static path `../Client/Client/dist` → `../Client/dist`.
- `Server/Server.js` — removed `parseInfo` schema field **AND** the dead tree-sitter parse feeding it (issue #20): dropped the `node-tree-sitter`/`tree-sitter-javascript`/`tree-sitter-python` requires, the `parseCode()` function, its call, and the `parseInfo:` store. (Extended beyond the minimal spec, which only removed the schema field — justified by #20's objective "eliminate the redundant tree-sitter parse.")
- Verified: `node --check Server.js` passes; `vite build` from flattened `Client/` passes (489 modules); `npm run dev` boots → http://localhost:5173.
- NOTE: `Server/package.json` still lists the three tree-sitter deps — code no longer uses them. Left installed (removing native-module deps is out of scope). Optional future cleanup.

### Post-phase env fix (not a plan phase)
- **Created `Server/.env`** (gitignored — NOT committed) with a generated 96-char random `JWT_SECRET`, plus `PORT`/`MONGO_URI`/`DOC_BUILDER_URL`/`NODE_ENV` from `.env.example`. Without it the Phase-1 hardening exits with `FATAL: JWT_SECRET…`. Server now boots past the JWT gate (then waits on Mongo at `localhost:27017`).
- **nvm-for-Windows notes** (env, not code): `npm run dev` error `Could not determine Node.js install directory` is transient — happens when `npm` runs while `nvm use` is recreating the `C:\nvm4w\nodejs` symlink; settles on its own. The "random cmd window" on `nvm use` is nvm-windows' `elevate` helper (benign).

---

## 5. Key file paths

```
DocuAgent/
├─ Server/
│  ├─ Server.js                     # Node gateway (Phases 0,1,2 edits)
│  ├─ .env.example                  # NEW (Phase 0)
│  └─ package.json                  # express 5, multer 2, mongoose 8
├─ Service/
│  ├─ AiAgent.py                    # Gemini -> markdown (Phase 0,4 edits)
│  ├─ Docbuilder.py                 # DOCX/PDF builder (Phase 4 edits)
│  ├─ Uml.py                        # diagram service (untouched)
│  ├─ requirements.txt              # trimmed (Phase 4)
│  └─ .env.example                  # NEW (Phase 0)
├─ Client/                          # FLATTENED in Phase 6 (was Client/Client/)
│  ├─ vite.config.js                # proxy target (Phase 2)
│  └─ src/
│     ├─ UploadPage.jsx             # Phases 2,3,4,5
│     ├─ HistoryPage.jsx            # Phases 2,5 (logout + empty-state border)
│     ├─ LoginPage.jsx              # Phase 2
│     ├─ SignupPage.jsx             # Phase 2
│     ├─ LandingPage.jsx            # Phase 2
│     ├─ App.jsx / main.jsx         # untouched
├─ .gitignore                       # NEW (Phase 0)
├─ README.md                        # Phase 0
├─ DocuAgent_Project_Plan.docx      # the plan  (DO NOT push)
├─ flags.md                         # issue registry (DO NOT push — lists unpatched vulns)
└─ reference.md                     # this file
```

---

## 6. NEXT SESSION — start here

**All 6 plan phases are DONE and committed.** Remaining actions:

1. **`git push`** the Phase 6 commit `07fe2b3` (verified safe — no secrets/internal docs in the diff). This is the only outstanding git action.
2. **Live end-to-end still unverified** — no phase was exercised through a real login → generate → download flow. That needs Client (`localhost:5173`) + Server (`3001`, needs `Server/.env`) + MongoDB (`27017`) + Gemini/UML Python services all running together. Recommended next verification step.

### Optional follow-ups (not in the 6-phase plan)
- Remove the three now-unused tree-sitter deps from `Server/package.json` (code no longer references them after Phase 6).
- Cross-check `flags.md` for any lower-priority issue outside the plan's scope that may still be open.
- `generatedFiles.pptx` field remains in `HistorySchema` though PPTX was dropped in Phase 4 — harmless dead field.

---

## 7. Housekeeping reminders
- **Internal docs stay OUT of git:** `DocuAgent_Project_Plan.docx`, `flags.md`, `reference.md` are internal-only and currently untracked — do NOT `git add` them. (`flags.md` lists unpatched vulns; the plan/reference are working notes.)
- All 6 phases committed (`568ed21`→`07fe2b3`). Phase 6 (`07fe2b3`) is **committed but not pushed** — `git push` when ready.
- Full end-to-end (`localhost:5173` login→generate→download) still unverified — needs Client + Server + MongoDB + Gemini/UML services all running together.
- Making `JWT_SECRET` mandatory means `Server/.env` MUST set it or the server won't boot (intended). **`Server/.env` was created this session** (gitignored) with a random secret — the server boots.
- Client now lives at `Client/` (flattened). Run the dev server with `cd Client && npm run dev` (→ `localhost:5173`); if you hit `Could not determine Node.js install directory`, it's a transient nvm-windows symlink glitch — just re-run.
