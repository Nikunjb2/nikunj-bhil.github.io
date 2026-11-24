# Mobile-First Dev Setup — System Design (v0, "mark 0")

**Goal:** Do real engineering work (write code, run experiments, write system designs, build apps/frameworks) **from a phone**, while borrowing compute from **high-end machines** (desktop, GPU box, cloud VM) when needed.

Working name: **PocketForge** (placeholder; rename freely).

---

## 1. Requirements

### Functional
| # | Requirement |
|---|-------------|
| F1 | Browse, edit, create files in a project from the phone |
| F2 | Full terminal (shell, git, build, run tests) |
| F3 | Run code on the phone for small experiments, and on remote machines for heavy work (builds, GPUs, Docker) |
| F4 | Connect to *multiple* machines and switch between them |
| F5 | Live preview of web apps / servers running on a remote machine |
| F6 | AI coding agent: describe a change by text/voice → review a diff → approve |
| F7 | Long-running jobs (training, builds) keep running when the phone sleeps; get notified when done |
| F8 | Write and render docs/diagrams (system designs, Markdown, Mermaid) |
| F9 | Git workflow: branch, commit, push, PR |

### Non-functional
- **Mobile-first UX:** thumb-reachable controls, minimal typing, works with on-screen keyboard.
- **Resilient to bad networks:** reconnect without losing terminal state or unsaved edits.
- **Secure:** no machine port exposed to the public internet; secrets stay on the machines, not the phone.
- **Low latency:** keystroke echo < 100 ms on good networks.
- **Battery friendly:** heavy work is always remote; phone is a thin-ish client.
- **Incremental:** usable on day 1 with off-the-shelf tools; custom pieces replace them one by one.

---

## 2. High-level architecture

```
          ┌──────────────────────── PHONE ────────────────────────┐
          │  PocketForge Client (PWA → later native wrapper)      │
          │  ┌─────────┐ ┌──────────┐ ┌────────┐ ┌─────────────┐  │
          │  │ Editor  │ │ Terminal │ │Preview │ │ AI / Diffs  │  │
          │  │(CM6)    │ │(xterm.js)│ │(iframe)│ │ review      │  │
          │  └────┬────┘ └────┬─────┘ └───┬────┘ └──────┬──────┘  │
          │       └──── Session Manager (multiplexed WS) ┘         │
          │   Local runtime (optional): Pyodide / JS / OPFS / git  │
          └─────────────────────────┬─────────────────────────────┘
                                    │  WireGuard mesh (Tailscale)  
                                    │  or relay tunnel fallback    
        ┌───────────────────────────┼──────────────────────────────┐
        │                           │                              │
 ┌──────▼───────┐           ┌───────▼──────┐               ┌───────▼──────┐
 │ Node Agent   │           │ Node Agent   │               │ Node Agent   │
 │ Desktop/WSL  │           │ GPU box      │               │ Cloud VM     │
 │ fs·pty·lsp   │           │ jobs·docker  │               │ always-on    │
 │ git·ports·AI │           │ ...          │               │ ...          │
 └──────────────┘           └──────────────┘               └──────────────┘
                                    │
                         ┌──────────▼──────────┐
                         │ Control plane (tiny)│  device registry, settings sync,
                         │ (optional, later)   │  job queue, push notifications
                         └─────────────────────┘
```

Three building blocks:
1. **Client** — mobile-first IDE UI.
2. **Node Agent** — one small daemon you install on every machine you want to use.
3. **Network + Control plane** — how they find and trust each other.

---

## 3. Components

### 3.1 Client (phone)
- **Form:** PWA first (installable, no app store, works on Android/iOS). Later wrap with Capacitor for native keyboard, background notifications, biometrics.
- **Editor:** **CodeMirror 6** — far better touch + mobile keyboard support than Monaco; small bundle; extensible.
- **Terminal:** **xterm.js** with a custom **extra-key bar**: `Esc Tab Ctrl Alt ↑ ↓ ← → | / - ~ { } [ ]`.
- **Layout:** single focused pane + bottom tab bar (Files · Editor · Terminal · Preview · AI). Swipe between panes; split view on tablets/landscape. Use the `visualViewport` API so panels don't hide behind the keyboard.
- **Input helpers:** command palette, snippets, voice dictation → AI, "select by tap on AST node" (expand selection to word → expression → block).
- **Offline cache:** open files + unsaved edits in IndexedDB/OPFS; replayed when reconnected.
- **Local runtime (optional):** Pyodide (Python), plain JS, isomorphic-git — for tiny experiments with no machine online.

### 3.2 Node Agent (each machine)
Single static binary (**Go** recommended: easy cross-compile to Linux/macOS/Windows/ARM, great PTY + WebSocket libraries). Runs as a user service.

| Service | Responsibility |
|---------|----------------|
| `fs` | list/read/write/watch files, search (ripgrep) |
| `pty` | create terminal sessions; **sessions survive disconnects** (ring buffer of output, tmux-like) |
| `proc` | run commands/jobs, stream logs, exit status |
| `lsp` | spawn language servers (pyright, tsserver, gopls, rust-analyzer) and proxy them |
| `git` | status/diff/commit/push helpers (wraps git CLI) |
| `ports` | detect listening dev servers, proxy them for the Preview pane |
| `jobs` | long-running tasks with persistence + completion notifications |
| `ai` | run a coding agent (e.g. Claude Code CLI / Agent SDK) inside the project; emit diffs for approval |
| `sys` | CPU/GPU/RAM/disk stats so the client can pick the right machine |

### 3.3 Protocol
- One **WebSocket per machine**, multiplexed into channels: `fs.*`, `pty.<id>`, `lsp.<id>`, `proc.*`, `ai.*` …
- Messages: JSON-RPC 2.0 for requests; binary frames for PTY data.
- **Resume:** every stream has a sequence number. On reconnect the client sends `resume {session, lastSeq}` and the agent replays the gap. Mobile networks switch Wi‑Fi↔LTE constantly — this is the most important reliability feature.
- Versioned (`hello {protocolVersion, capabilities}`) so old agents keep working.

### 3.4 Networking & security
- **Phase 1:** **Tailscale** (WireGuard mesh). Every device gets a private IP; nothing exposed publicly; works through NAT. Zero code.
- **Fallback:** relay tunnel (Cloudflare Tunnel or own relay) for networks that block UDP.
- **Auth:** device pairing via QR code (agent prints QR → phone scans → exchanges keys). Each request signed with the device key; mTLS on the WebSocket. Optional passkey/biometric unlock in the client.
- **Secrets** (API keys, SSH keys, cloud creds) live only on agents. The phone holds only its device key.
- **Least privilege:** each agent restricts access to configured workspace roots; AI agent runs with an allow-list of commands.

### 3.5 Control plane (later, optional)
Small service (or just a git repo + Tailscale at first):
- device + machine registry, health
- settings/snippets/keybindings sync
- job queue: "run this on whichever GPU machine is free"
- push notifications (Web Push / FCM) when a job/AI task finishes

---

## 4. Key flows

**Edit & run**
1. Phone opens project on machine `desktop` → `fs.list`, `fs.read`.
2. Edits buffered locally, saved via `fs.write` (debounced, with version check to avoid clobbering).
3. Terminal tab → `pty.open` → `npm test`. Phone locks → PTY keeps running. Unlock → `resume` replays output.

**AI-driven change (the core mobile workflow)**
1. Voice/text: "add pagination to /users endpoint".
2. Client → `ai.task` on the agent; agent runs the coding agent in the repo.
3. Agent streams progress; result = a **diff**.
4. Phone shows a swipeable per-file diff: approve / reject / comment → agent iterates.
5. Approve → commit + push → PR.

> Typing code on a phone is slow; *reviewing* code on a phone is fine. The design centres on review-and-approve.

**Heavy experiment**
1. `jobs.submit {machine: "gpu-box", cmd: "python train.py"}`.
2. Phone closes. Agent finishes → push notification with summary + metrics link.

---

## 5. Tech stack (recommended)

| Layer | Choice | Why |
|-------|--------|-----|
| Client UI | TypeScript + Svelte/SolidJS (or React if you prefer) + Vite | small bundles = fast on phones |
| Editor | CodeMirror 6 | best mobile support |
| Terminal | xterm.js | standard |
| Packaging | PWA → Capacitor | ship fast, go native later |
| Agent | Go | single binary, cross-platform, ARM (even runs in Termux) |
| Network | Tailscale → own relay later | secure from day 1 |
| AI | Claude Code CLI / Claude Agent SDK on the agent | real repo access, runs where the compute is |
| Docs | Markdown + Mermaid rendered in client | for system designs |

---

## 6. Roadmap

### Phase 0 — Use it TODAY (no code)
- Install **Tailscale** on phone + every machine.
- On the high-end machine: **code-server** (VS Code in browser) or **SSH + tmux**.
- Phone: **Termux** (Android) for local shell/git/python; a good SSH client.
- **Claude Code on the web / mobile app** for AI tasks against your GitHub repos (what you're using now).
- Keep everything in GitHub so any device can pick up work.

This gives ~70% of the value immediately and shows what the custom build must improve.

### Phase 1 — MVP (≈2–4 weeks)
- Node Agent: `fs`, `pty` (with resume), `hello`, QR pairing.
- PWA: file tree, CodeMirror editor, terminal + extra-key bar, single machine.

### Phase 2 — Multi-machine IDE
- Machine switcher, `sys` stats, `ports` preview, `git` panel, LSP (diagnostics, completion, go-to-definition).

### Phase 3 — AI + jobs
- `ai.task` with diff review UI, `jobs` + push notifications, voice input.

### Phase 4 — Polish & extend
- Capacitor native app, offline local runtime, plugin API, settings sync via control plane, templates (system design, web app, CLI, library/framework).

---

## 7. Risks / trade-offs
- **iOS PWA limits** (background, keyboard quirks) → Capacitor wrapper in Phase 4.
- **Building an editor is huge** → don't; compose CodeMirror + LSP.
- **Scope creep** → Phase 0 tools cover gaps until each custom piece is clearly better.
- **Security of a remote shell** → never expose agents publicly; pairing + mTLS + Tailscale ACLs.

---

## 8. Open decisions (need your input)
1. Phone OS: Android, iOS, or both? (Android + Termux unlocks much more.)
2. Your high-end machines: OS (Windows/WSL, Linux, macOS)? GPU? Always on?
3. Main languages you'll work in (for which LSPs to ship first)?
4. Separate repo for PocketForge (recommended) vs. inside this site repo?
5. UI framework preference (Svelte / Solid / React)?
