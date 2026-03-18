# Tableau Next Demo Builder

Build complete, realistic Tableau Next demos end-to-end — no coding required.

You describe the prospect, the persona, and the story. Claude builds the workspace, semantic model, visualizations, and AI-ready metrics automatically in your Salesforce org.

---

## Before You Start

**Software (on your Mac):**
- [ ] [Claude Code](https://claude.ai/code) installed
- [ ] Python 3 installed — check by opening Terminal and typing `python3 --version`
- [ ] The `requests` Python package — install by typing `pip3 install requests` in Terminal

**Salesforce:**
- [ ] A Salesforce org with Data Cloud enabled
- [ ] Admin access (or someone who can create a Connected App for you)

---

## Step 1 — One-Time Setup

Open Terminal, then drag this folder into the Terminal window and press Enter to navigate into it. Then run:

```
python3 next_setup.py
```

The script walks you through everything — including step-by-step instructions for the two things you need to do in the Salesforce UI (creating a Connected App and an IngestAPI connector). Takes about 10–15 minutes.

When it finishes, a file called `next_config.json` will appear in this folder. Keep it private — it contains your credentials. Do not share it.

**You only run setup once per Salesforce org.**

---

## Step 2 — Open This Folder in Claude Code

1. Open VS Code
2. Go to **File → Open Folder** and select this folder
3. Click the **Claude Code** icon in the bottom status bar to open the Claude panel

If you don't see the Claude Code icon, install the Claude Code extension from the VS Code Extensions marketplace first.

Once the folder is open, type **hi** in the Claude panel to confirm everything is working — Claude will introduce itself and explain what it can do.

---

## Step 3 — Build a Demo

Type:
```
/build-next-demo
```

Claude will ask for:
- The prospect's name (e.g., "First Meridian Bank")
- The persona (e.g., "Commercial Banking Relationship Manager")
- The story (e.g., "Loan originations have been declining for the past 6 months")

It will show you a plan and ask for your approval before building anything. Once you say go, it handles everything automatically — generating data, loading it into Data Cloud, building the semantic model, and creating the visualizations.

The build takes about 5–10 minutes. When it finishes, your demo is live in Salesforce and a demo guide is written to this folder.

---

## What Gets Built

For each demo, Claude creates:
- **Synthetic data** — realistic financial data with an engineered signal (e.g., a regional decline that started 6 months ago)
- **Data Cloud schemas and streams** — registers the data structure and loads the data into your Salesforce org automatically
- A **Workspace** in Tableau Next
- A **Semantic Data Model** with related tables, calculated metrics, and relationships
- **Sample visualizations** — a starter set of charts built on the model (you can build more in the UI)
- **AI-ready metrics** configured for Concierge (the natural language Q&A feature)
- A **demo guide** with suggested Concierge questions and a run-of-show

---

## Cleaning Up After a Demo

To remove a demo from your org after a meeting:

```
python3 next_teardown.py
```

It will show you a list of demos you've built and walk you through the cleanup.

---

## Common Issues

**"ModuleNotFoundError: No module named 'requests'"**
Run `pip3 install requests` in Terminal and try the setup script again.

**Setup script says port 8080 is already in use**
Something else on your machine is using that port. Quit other applications (Zoom, local web servers) and try again.

**Something else went wrong**
Paste the error message into Claude Code and ask what to do. Claude can diagnose and fix most issues automatically.
