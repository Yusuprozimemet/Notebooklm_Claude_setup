# Claude + NotebookLM Setup & Usage Guide

> A practical guide based on real installation experience on Windows.  
> Skill source: https://github.com/PleasePrompto/notebooklm-skill

---

## Next Steps (Quick Start)

Follow these 4 steps to go from zero to your first query.

**Step 1 — Authenticate with Google (one-time setup)**

This will open a visible Chrome window for you to log in manually.

```bash
cd ~/.claude/skills/notebooklm
PYTHONIOENCODING=utf-8 py scripts/run.py auth_manager.py setup
```

**Step 2 — Create a NotebookLM notebook**

1. Go to [notebooklm.google.com](https://notebooklm.google.com)
2. Create a notebook and upload your documents
3. Share it: **Share > Anyone with the link**

**Step 3 — Add it to your library (in Claude Code)**

```
"Add this NotebookLM to my library: [your-link]"
```

Claude will auto-discover the notebook's content and register it.

**Step 4 — Start querying**

```
"Ask my [notebook name] about [topic]"
```

---

## What This Gives You

- **NotebookLM** — Gemini-powered answers grounded exclusively in your uploaded documents (zero hallucination on in-scope topics)
- **Claude Code** — builds code, synthesizes decisions, writes files, manages your project
- **Together** — Claude queries your knowledge base at each decision point, grounding every recommendation in your own docs

---

## Part 1 — Install the Skill

### 1.1 Clone into the skills directory

```bash
cd ~/.claude/skills
git clone https://github.com/PleasePrompto/notebooklm-skill notebooklm
```

### 1.2 Create the virtual environment manually (Windows)

The auto-setup script has emoji encoding issues on Windows. Do this instead:

```bash
cd ~/.claude/skills/notebooklm
py -m venv .venv
```

### 1.3 Install dependencies directly

```bash
~/.claude/skills/notebooklm/.venv/Scripts/pip.exe install patchright==1.55.2 python-dotenv==1.0.0
```

> Chrome installation: patchright uses the Chrome already installed on your system.  
> If you see "ATTENTION: chrome is already installed" — that's fine, not an error.

### 1.4 Verify the venv exists

```bash
ls ~/.claude/skills/notebooklm/.venv/Scripts/python.exe
```

If the file exists, setup is complete.

---

## Part 2 — Authenticate

### 2.1 Check if already authenticated

```bash
cd ~/.claude/skills/notebooklm
PYTHONIOENCODING=utf-8 py scripts/run.py auth_manager.py status
```

### 2.2 First-time login (if not authenticated)

```bash
PYTHONIOENCODING=utf-8 py scripts/run.py auth_manager.py setup
```

A Chrome browser window will open. Log in to the Google account that owns your NotebookLM notebooks. The session is saved — you won't need to log in again.

> **Windows note**: Always prepend `PYTHONIOENCODING=utf-8` to every skill command.  
> Without it, emoji characters in script output will crash with a `UnicodeEncodeError`.

---

## Part 3 — Add a Notebook

NotebookLM notebooks must be shared with link access (not private).

### 3.1 Smart Add (recommended — auto-discovers content)

Step 1: Query the notebook to discover what it contains:

```bash
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py \
  --question "What is the content of this notebook? What topics are covered? Give a complete overview." \
  --notebook-url "https://notebooklm.google.com/notebook/YOUR-NOTEBOOK-ID"
```

Step 2: Register it using what you learned:

```bash
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/YOUR-NOTEBOOK-ID" \
  --name "Descriptive Name" \
  --description "What this notebook contains" \
  --topics "topic1,topic2,topic3"
```

### 3.2 List all notebooks

```bash
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py list
```

### 3.3 Set active notebook

```bash
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py activate --id YOUR-NOTEBOOK-ID
```

---

## Part 4 — Query Your Notebook

```bash
# Query active notebook
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py --question "Your question here"

# Query specific notebook by ID
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py \
  --question "Your question" --notebook-id YOUR-ID

# Query by URL
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py \
  --question "Your question" \
  --notebook-url "https://notebooklm.google.com/notebook/..."

# Debug: show browser window
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py \
  --question "Your question" --show-browser
```

---

## Part 5 — Collaborative Workflow with Claude

### How it works

Claude triggers the skill automatically when you mention NotebookLM in conversation. You can also reference it explicitly: *"check my docs about X"* or *"ask my NotebookLM about Y"*.

### Workflow for building an app

1. **Prepare your knowledge base** — upload your planning docs, PRD, API references, or ADRs to NotebookLM. Share the notebook (anyone with link).

2. **Add it to the library** — use Smart Add above.

3. **Start a Claude Code session** in your project directory.

4. **Reference your docs at decision points**:

   > "Before choosing a database, ask my NotebookLM what our data model requirements are."

   > "Check my docs — what does our PRD say about real-time requirements?"

   > "Query my architecture notebook — have we decided on auth strategy?"

5. **Claude synthesizes** — takes NotebookLM's grounded answer + its own engineering knowledge and makes a concrete recommendation.

### What NotebookLM handles well

| Use Case | How to trigger |
|---|---|
| Requirements lookup | "What does my PRD say about X?" |
| Architecture decisions | "What approach did we decide for Y?" |
| API reference | "What are the parameters for endpoint Z?" |
| Design rationale | "Why did we choose approach A over B?" |
| Scope confirmation | "Is feature X in scope for v1?" |

### What Claude handles

- Writing, editing, and reviewing code
- Running tests and analyzing failures
- Synthesizing NotebookLM answers into concrete decisions
- Tracking implementation progress
- Flagging when a decision contradicts stated requirements

---

## Part 6 — Task-Specific Usage Examples

### System Design Sessions

Upload your requirements doc to NotebookLM. Then in Claude:

> "We're designing a URL shortener. Check my requirements notebook — what are the stated NFRs for availability and latency?"

Claude queries NotebookLM, gets grounded NFRs, then applies the system design framework anchored to your actual requirements.

### Code Review Against Spec

Upload your API spec or architecture doc. Then:

> "Review this implementation against my API spec notebook — does it match the contract?"

### Onboarding a New Codebase

Upload README, architecture diagrams, and runbooks. Then:

> "I'm about to modify the auth middleware — check my docs for any notes on the current auth design."

### Decision Log Reference

Upload your ADR (Architecture Decision Records) doc. Then:

> "Before I change the caching strategy, ask my ADR notebook if there's a record for this decision."

---

## Limits to Know

| Limit | Details |
|---|---|
| Queries per day | ~50 on free Google account |
| Notebook visibility | Must be shared with link (not private) |
| Session context | Each query is independent — include context in each question |
| Code writing | NotebookLM answers questions; it does not write code |
| File size | Large documents may be truncated by NotebookLM |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `UnicodeEncodeError` on Windows | Prepend `PYTHONIOENCODING=utf-8` to every command |
| `ModuleNotFoundError` | Never call scripts directly — always use `py scripts/run.py` |
| `python` not found | Use `py` on Windows (the Python launcher) |
| Auth fails | Browser must be visible: `auth_manager.py setup` opens Chrome automatically |
| Notebook not found | Run `notebook_manager.py list` to see registered IDs |
| Stale session | Run `auth_manager.py reauth` |
| Browser crashes | Run `cleanup_manager.py --preserve-library` to reset browser state |

---

## Quick Reference

```bash
# All commands — run from: ~/.claude/skills/notebooklm

# Auth
PYTHONIOENCODING=utf-8 py scripts/run.py auth_manager.py status
PYTHONIOENCODING=utf-8 py scripts/run.py auth_manager.py setup

# Notebooks
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py list
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py add --url URL --name NAME --description DESC --topics TOPICS
PYTHONIOENCODING=utf-8 py scripts/run.py notebook_manager.py activate --id ID

# Query
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py --question "..." 
PYTHONIOENCODING=utf-8 py scripts/run.py ask_question.py --question "..." --notebook-id ID
```
