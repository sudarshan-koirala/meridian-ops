# Meridian Ops: a team of AI agents on Databricks

Meridian Labs is a made-up software company. It is losing customers in the APAC region, and nobody knows why.

This project builds a small AI team that finds the answer:

- **Ops Lead** (the main agent): reads your question, asks the right helpers, and writes one short report.
- **Accounts Analyst**: answers questions about customers, plans, regions, monthly revenue and lost customers.
- **Support Analyst**: answers questions about support tickets, reply times and customer ratings.
- **Policy Reader**: runs a little Python code to read the company's support policy document.

You ask one question. The Ops Lead calls all three helpers and tells you what to fix first.

```text
Why are we losing customers in APAC? Check accounts, support tickets and our SLA policy, then tell me what to fix first.
```

## What is in this repo

| Folder or file | What it is |
| --- | --- |
| `data/meridian_accounts.csv` | 100 customer accounts (25 per region) |
| `data/meridian_support_tickets.csv` | 120 support tickets |
| `data/Meridian-Support-SLA-Policy.pdf` | The support policy (the promised reply times) |
| `data/Meridian-Support-SLA-Policy.md` | The same policy as plain text, so the agent can read it easily |
| `agent/agent.py` | The Ops Lead and its instructions (`SYSTEM_PROMPT`) |
| `agent.toml` | The list of helpers the Ops Lead can call |

## What you need

- A Databricks workspace. The free Free Edition works for trying it on your computer. To put it online (step 7), you need a trial or paid workspace.
- The Databricks command line tool. Check it with `databricks --version`.
- `uv` with Python 3.12. Install Python with `uv python install 3.12`.
- In your workspace: Genie turned on and one SQL warehouse running (2X-Small is enough on Free Edition).

## Step 1: Load the data

In your Databricks workspace, open a SQL editor and run:

```sql
CREATE SCHEMA IF NOT EXISTS mycatalog.meridian_labs;
CREATE VOLUME IF NOT EXISTS mycatalog.meridian_labs.policies;
```

Then, in **Catalog**:

1. Upload `data/meridian_accounts.csv` as a table named `meridian_accounts`.
2. Upload `data/meridian_support_tickets.csv` as a table named `meridian_support_tickets`.
3. Upload both policy files (`.pdf` and `.md`) into the `policies` volume.

If you already have older Meridian tables there, replace them with these files. Otherwise your numbers will not match the ones below.

> **Your catalog may not be called `mycatalog`.** Free Edition often uses `workspace`. If yours is different, change the name everywhere: in the SQL above, in `SYSTEM_PROMPT` in `agent/agent.py`, and in every `dbfs:/Volumes/...` path.

## Step 2: Build the two analysts with Genie

Create two Genie Agents in your workspace:

- **Accounts Analyst**, using the `meridian_accounts` table.
- **Support Analyst**, using the `meridian_support_tickets` table.

Test each one with a question:

```text
[Accounts Analyst] What is the churn rate by region? Show it as a percent.
[Support Analyst]  What is the average first response time in hours by region?
```

Copy the ID of each one from its **Settings** tab. It is a 32-character code of numbers and lowercase letters. You need both IDs in step 4.

## Step 3: Install the Agent Bricks tool and sign in

```bash
uv tool install --python 3.12 databricks-agentbricks
agentbricks --version

databricks auth login --host "https://<your-workspace-url>" --profile my-workspace
agentbricks login --profile my-workspace
```

If you are starting from scratch instead of from this repo, create the project with:

```bash
agentbricks init --framework langgraph meridian-ops
cd meridian-ops
```

## Step 4: Connect the helpers

Put your two Genie IDs from step 2 in place of the brackets:

```bash
agentbricks tools add genie-agent [ACCOUNTS_SPACE_ID] --name accounts_analyst
agentbricks tools add genie-agent [SUPPORT_SPACE_ID] --name support_analyst
agentbricks tools add sandbox --name policy_reader
```

This repo's `agent.toml` already lists these helpers, but with the IDs from the original workspace. Replace them with yours.

Do **not** add `--scope` or `--permission` to the sandbox command. It does not support them, and the agent will not start. The sandbox uses your own access rights.

## Step 5: Tell the Ops Lead how to work

The instructions live in `SYSTEM_PROMPT` in `agent/agent.py`. They are already filled in:

```text
You are the Meridian Labs Ops Lead. You manage a team of specialists.
Never guess numbers. Always ask a specialist.

Your team:
- accounts_analyst: customers, tiers, regions, MRR and churn.
- support_analyst: support tickets, first response times and CSAT.
- sandbox (create_sandbox / run_code / delete_sandbox): runs Python to read policy files.
  /Volumes is NOT mounted in the sandbox. Do not use open() or os.listdir() on /Volumes.
  Read the SLA policy with the Databricks CLI, which is already signed in:
    import subprocess
    r = subprocess.run(["databricks", "fs", "cat",
        "dbfs:/Volumes/mycatalog/meridian_labs/policies/Meridian-Support-SLA-Policy.md"],
        capture_output=True, text=True)
    print(r.stdout or r.stderr)
  Call delete_sandbox when you are done.

Rules:
1. If a question touches more than one area, call every specialist you need.
2. Combine their answers into one short report.
3. Say which specialist gave which fact.
4. End with the top 3 actions, most important first.
```

## Step 6: Try it on your computer

```bash
agentbricks dev
```

Open http://localhost:8000 and ask:

```text
Which region has the highest churn?
```

```text
Why are we losing customers in APAC? Check accounts, support tickets and our SLA policy, then tell me what to fix first.
```

Follow-up:

```text
Which APAC accounts had two or more SLA breaches? They should be on the churn-risk watch list.
```

The first run is slow because it installs everything. Later runs are fast.

### What you should see

- **Lost customers:** APAC 40.0%. North America, Europe and LATAM 12.0% each.
- **Average first reply:** APAC 11.9 hours. North America 2.8, Europe 2.8, LATAM 2.5.
- **Customer rating (1 to 5):** APAC 2.16. Other regions 3.96 to 4.30.
- **First replies later than promised:** APAC 80.0%. All other regions 0.0%.
- **The reason**, from section 3 of the policy: APAC tickets go to the Europe team. That team starts at 08:00 Central European Time. There is no APAC team until Q2 2027.

## Step 7: Put it online and share it

This step needs a trial or paid workspace. It does not work on Free Edition. If you built steps 1 to 4 on Free Edition, sign in to the other workspace with `agentbricks login --profile <profile>` and repeat steps 1 to 4 there. Keep your `agent/agent.py`.

```bash
agentbricks deploy meridian-ops
agentbricks deployments list
```

The name can be at most 17 characters. `meridian-ops` has 12.

## If the answer is not right

**The Ops Lead does not ask all three helpers:**

```text
Use accounts_analyst, support_analyst and the sandbox to explain why APAC churn is high. Read the SLA policy in the sandbox. Call all three before you answer.
```

**The Policy Reader cannot find or read the policy:**

```text
Use the sandbox. Run the Databricks CLI command: databricks fs cat dbfs:/Volumes/mycatalog/meridian_labs/policies/Meridian-Support-SLA-Policy.md and tell me what section 3 says about APAC.
```

Also check that the `.md` file is in the volume:

```bash
databricks fs ls dbfs:/Volumes/mycatalog/meridian_labs/policies/
```

**Genie gives averages instead of percents:**

```text
What percent of accounts have status churned, grouped by region? Round to one decimal.
```

**You want a shorter question:**

```text
What is going wrong in APAC, and what should we fix first?
```

## Common problems

| Problem | Fix |
| --- | --- |
| Install fails on an Intel Mac (`cryptography`) | `pip install --only-binary :all: databricks-agentbricks` |
| `ToolManifestError ... declares policy.downscope` | Open `agent.toml`. Delete the `policy` / `downscope` lines from the sandbox block. Run `agentbricks dev` again. |
| `[Errno 95] Operation not supported` on `/Volumes` | The sandbox cannot open volume files directly. The instructions in step 5 must tell it to use `databricks fs cat dbfs:/Volumes/...`. |
| The sandbox finds no policy file | Run `databricks fs ls dbfs:/Volumes/<catalog>/meridian_labs/policies/` from your own terminal. If it is empty, upload the files with `databricks fs cp`. |
| Memory shows "Not connected" during `agentbricks dev` | This is normal. Memory only connects after step 7. |
| Genie says you are asking too often (Free Edition) | Wait one minute between big questions. |
| Putting it online fails with `Scale-to-zero must be enabled for this workspace tier` | Free Edition does not allow this. Use a trial or paid workspace. |
| Putting it online fails because of memory or saved chats | Run `agentbricks memory unbind && agentbricks sessions unbind`, then try again. |
| Free Edition says you have too many apps (the limit is 3) | Delete the failed one: `agentbricks deployments delete agent-bricks-meridian-ops` |

## For developers

The details of how the code fits together are in [AGENTS.md](AGENTS.md) and [AGENTKIT_CONTRACT.md](AGENTKIT_CONTRACT.md). Run the tests with `uv run pytest`.
