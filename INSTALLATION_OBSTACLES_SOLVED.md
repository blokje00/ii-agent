# ii-agent Installation – Obstacles Solved (M1 Mac)

A step-by-step checklist of every problem encountered while bringing the **ii-agent** repository up on an Apple-Silicon MacBook, together with the exact fix and a “prevention” tip for future installs.

---

## 1  Git / Repository State
| # | Obstacle | What We Saw | Fix (commands) | Prevent Next Time |
|---|----------|-------------|----------------|-------------------|
|1.1|Untracked change in `frontend/yarn.lock` blocked clean install|`git status` showed modified file|``git add frontend/yarn.lock && git commit -m "Update yarn.lock"``|Run `git status` before install; commit or stash leftover edits.|

---

## 2  Toolchain & Environment
| # | Obstacle | What We Saw | Fix (commands) | Prevent Next Time |
|---|----------|-------------|----------------|-------------------|
|2.1| Correct Python version missing | `python --version` absent; system Python 3.13 installed, project requires ≥3.10 | a) Verify: `pyenv install --list | grep 3.10`<br>b) (Optional) `pyenv install 3.10.17 && pyenv global 3.10.17`<br>c) Used `python3 -m venv .venv` | Keep **pyenv** installed; specify `pyenv local 3.10.x` in repo for deterministic builds. |
|2.2| Xcode Command Line Tools not guaranteed | Some wheels compile native code | `xcode-select --install` (no output means already installed) | Verify once: `xcode-select -p` in bootstrap script. |
|2.3| Playwright browsers missing | Runtime errors from tools that open a browser | `playwright install --with-deps` inside venv | Add post-install script: `pip install -e . && playwright install`. |

---

## 3  Virtual Environment & Dependency Installation
| # | Obstacle | What We Saw | Fix (commands) | Prevent Next Time |
|---|----------|-------------|----------------|-------------------|
|3.1| venv absent | Import errors | `python3 -m venv .venv && source .venv/bin/activate` | Add `make venv` target to repo. |
|3.2| Heavy dependency set on M1 | Build errors possible (Pillow, PyMuPDF) | Homebrew libs already present; ensured `brew install jpeg zlib` when needed | Document M1 Homebrew prerequisites in README. |

---

## 4  Environment Variables / Secrets
| # | Obstacle | What We Saw | Fix (steps & cmds) | Prevent Next Time |
|---|----------|-------------|--------------------|-------------------|
|4.1| `.env` file missing | Backend crashed on startup | Created root `.env` with required keys (`ANTHROPIC_API_KEY`, `STATIC_FILE_BASE_URL`, etc.) | Ship `.env.example`. |
|4.2| API key kept “disappearing” | After edits, file reverted to placeholder | a) Found file was overwritten during edits.<br>b) Set correct key, then locked file: `chmod 444 .env`.<br>c) Backup: `cp .env .env.backup` | Use `chmod 400 .env` in CI; never open with tools that rewrite on save. |
|4.3| Frontend couldn’t reach backend | 404 on fetch | Added `frontend/.env` with<br>`NEXT_PUBLIC_API_URL=http://localhost:8000` | Keep example in repo; validate on `npm run dev`. |

---

## 5  Process Management
| # | Obstacle | What We Saw | Fix | Prevent Next Time |
|---|----------|-------------|-----|-------------------|
|5.1| Backend (FastAPI/Uvicorn) not running | No response on :8000 | `python ws_server.py --port 8000` (wrapped in `nohup … &`) | Provide `make backend`. |
|5.2| Wrong Node process (from another repo) on :3000 | Blank page | `pkill -f "npm run dev"` then start correct one: `npm run dev` in `frontend/` | `npx kill-port 3000` before launch. |
|5.3| Need clean restarts after config changes | Stale env vars | Killed by `kill -TERM <pid>` and relaunched | Add `make restart` shortcut. |

---

## 6  Frontend Hydration Error
| # | Obstacle | What We Saw | Fix (code) | Prevent Next Time |
|---|----------|-------------|------------|-------------------|
|6.1| React hydration error: `<button>` inside `<button>` | Browser console: “In HTML, `<button>` cannot be a descendant of `<button>`” | a) Updated `TooltipTrigger` to accept `asChild`.<br>b) Wrapped `Button` with `<TooltipTrigger asChild>` in `components/question-input.tsx`.<br>c) Added doc-comment to `Button` component. | Run `npm run lint` with accessibility plugin; add hydration-test page to CI. |

---

## 7  Security & Stability Hardening
| Measure | Detail |
|---------|--------|
|Lock secrets| `.env` set to read-only (`chmod 444`) after inserting keys. |
|Log isolation| Backend logs piped to `ws_server.log`; frontend to `frontend.log`. |
|Background services | Launched via `nohup … &` for persistence across terminal closure. |

---

## 8  Final Verification
1. **Backend health**: `curl -s http://localhost:8000/api/sessions/health` returns `{"sessions":[]}`  
2. **Frontend**: Browse to `http://localhost:3000` – page loads, no console errors.  
3. **Agent call**: Start a chat; request processed by Anthropic using new key.  

---

### Quick Re-run Script (future installs)

```bash
# prerequisites: brew, pyenv, node 18, yarn, xcode-tools, playwright deps
git clone <repo> && cd ii-agent

pyenv install -s 3.10.17 && pyenv local 3.10.17
python -m venv .venv && source .venv/bin/activate
pip install -e . && playwright install

cp .env.example .env
echo "ANTHROPIC_API_KEY=sk-..." >> .env
chmod 444 .env

( cd frontend && npm install )

# start
source .venv/bin/activate && nohup python ws_server.py --port 8000 &
( cd frontend && npm run dev )
```

Keep this checklist to speed-run future setups or debug similar workstations.
