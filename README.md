# 🤖 ZenTrades AI — Automated Call Processing Pipelines

> *"So you've got 10 call transcripts, a Groq API key, and a dream. Let's build something."*

This repo automates the boring part of onboarding trade businesses (HVAC, plumbing, pest control, and friends). Drop in a call transcript, and these n8n pipelines will spit out a structured account memo and a fully configured AI agent spec — no copy-pasting, no crying.

Built by **Aryaman** for the ZenTrades AI assignment. It works. Please don't touch the JavaScript nodes.

---

## 📂 Folder Structure

Before you run anything, your folder layout needs to look *exactly* like this. The pipelines are picky. They will not ask nicely if a file is missing.

```text
ZenTrades AI/
├── inputs/
│   ├── demo/
│   │   ├── demo_001_arctic_hvac.txt
│   │   ├── demo_002_riverstone_plumbing.txt
│   │   └── ... (up to 005)
│   └── onboarding/
│       ├── onboarding_001_arctic_hvac.txt
│       ├── onboarding_002_riverstone_plumbing.txt
│       └── ... (up to 005)
└── outputs/
    ├── accounts/
    │   ├── ACC-001_memo_v1.json
    │   ├── ACC-001_memo_v2.json
    │   └── ...
    └── agent_spec/
        ├── 001_arctic_hvac_agent_spec_v1.json
        └── ...
```

> 💡 The `outputs/` folder gets created automatically when the pipelines run. You only need to set up `inputs/` manually.

---

## 🏷️ File Naming — This Actually Matters

The JavaScript nodes parse filenames to figure out which account they're working with. Deviate from this format and things will break silently. You've been warned.

| File Type | Format | Example |
|---|---|---|
| Demo transcript | `demo_[id]_[company].txt` | `demo_001_arctic_hvac.txt` |
| Onboarding transcript | `onboarding_[id]_[company].txt` | `onboarding_003_fireguard.txt` |
| Account memo output | `ACC-[id]_memo_[version].json` | `ACC-001_memo_v1.json` |
| Agent spec output | `[id]_[company]_agent_spec_[version].json` | `001_arctic_hvac_agent_spec_v1.json` |

---

## ⚙️ What You Need Before Starting

Two things. Just two.

**1. A running n8n instance**

Locally via Docker is the way to go here, because the pipelines read and write actual files from disk. Cloud n8n won't have access to your local folders (obviously). If you haven't set it up yet, [the Docker setup guide is here](https://docs.n8n.io/hosting/installation/docker/).

Make sure you mount your `ZenTrades AI` folder as a volume so n8n can see it inside the container:

```yaml
volumes:
  - "C:/Users/ARYAMAN/Downloads/ZenTrades AI:/data/zentrades"
```

Inside n8n, your paths will then start with `/data/zentrades/inputs/...` and `/data/zentrades/outputs/...`

**2. A Groq API Key**

Sign up at [console.groq.com](https://console.groq.com) — it's free. Grab your API key.

In n8n, find the HTTP Request nodes in both workflows and add your key as a header:

```
Authorization: Bearer YOUR_GROQ_API_KEY
```

That's it. No paid plans, no credit card, no tears.

---

## 🚀 Running the Pipelines

There are two workflows. They do different things. Run them in order.

---

### Pipeline 1 — Two-Stage Sequential Processing

**What it does:** Takes one input file, runs it through Groq *twice* (first to extract a v1 memo, then to build an agent spec from that memo), and saves both outputs to disk.

Think of it as: `raw transcript → structured memo → polished agent config`

**How to run it:**

1. Open the workflow in n8n
2. Make sure your target `.txt` file is sitting in the right `inputs/` subfolder
3. Click the first **Read/Write Files from Disk** node and hit **"Execute node"**
4. Watch the green checkmarks cascade rightward like dominoes
5. Check your `outputs/` folder — you should see two new files written by the `Disk1` and `Disk2` write nodes

> ⚠️ If a node turns red, click it and read the error. 90% of the time it's either a wrong file path or a missing API key.

---

### Pipeline 2 — Batch Processing (The Fun One)

**What it does:** Reads all 5 input files at once, fires 5 parallel requests to Groq, and writes all 5 output JSONs in one go. It's the same work as Pipeline 1, just done in bulk and significantly more satisfying to watch.

**How to run it:**

1. Make sure **all 5 files** (`001` through `005`) are in their respective `inputs/` folder — batch mode will look for all of them
2. Open the workflow in n8n
3. Click **"Execute Workflow"** at the bottom of the screen (or trigger the leftmost node)
4. Groq gets 5 requests, processes them, and the final **Read/Write Files from Disk5** node writes all 5 output JSONs into `outputs/` — file names are generated dynamically from the input file names, so no manual renaming needed

> 💡 If you're running both pipelines back to back, Pipeline 1 first, then Pipeline 2. The batch pipeline can depend on some outputs that Pipeline 1 generates.

---

## 🗂️ What Gets Generated

After both pipelines run successfully on all 10 files, your `outputs/` folder will contain:

- **10 account memo JSONs** — 5 at `v1` (from demo calls), 5 at `v2` (updated after onboarding)
- **10 agent spec JSONs** — ready to paste into Retell or review manually
- One less headache

---

## 🐛 Common Issues

**"File not found" errors**
Check your volume mount in `docker-compose.yml`. The path inside the container (`/data/zentrades/`) must match exactly what's in the n8n file path nodes. One wrong slash ruins everything.

**Groq returns a 429 error**
You're hitting the rate limit. Add a **Wait** node (set to 2 seconds) between the read and HTTP Request nodes in Pipeline 2.

**Outputs folder is empty after running**
The write nodes might have the wrong output path configured. Click the final `Read/Write Files from Disk` node in each pipeline and double-check the file path points to `/data/zentrades/outputs/`.

**JSON parsing fails in the Code node**
Groq occasionally wraps its response in markdown code fences (` ```json ... ``` `). The Code nodes already strip these — but if you're customizing prompts, make sure your system prompt says *"respond with raw JSON only, no markdown formatting."*

---

*Maintained by Aryaman · ZenTrades AI Assignment · Built with n8n + Groq + a concerning amount of coffee*