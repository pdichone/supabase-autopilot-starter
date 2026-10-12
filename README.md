# supabase-autopilot-starter

Run Supabase's **Health monitor** prompt every hour against one project, read-only, as a Claude Code routine. This is the companion repo to the video *Put your Supabase project on autopilot with Claude Routines + MCP*.

What you get:

- `.mcp.json`: the hosted Supabase MCP server, scoped to one project, read-only, with only the `docs`, `debugging` and `database` tool groups. No token in it. Locally you sign in with OAuth; in the routine, a network secret on the cloud environment supplies the token.
- `prompts/`: Supabase's four monitoring prompts (Health, Security, Performance, Resources), copied verbatim with source links.
- `routine-prompt.md`: the Health prompt plus a few lines that tell Claude where its state lives in this repo. Paste this into the routine.
- `state/health.example.json`: what the saved state looks like after a run.
- `CLEANRUN.md`: the three checks that prove the setup works from a fresh clone.
- `.claude/settings.json`: pre-approves the project MCP server and the tools the prompt uses (the four read-only Supabase tools, fetching `supabase.com`, writing under `state/`, and git), so the on-demand run doesn't stop for permission prompts. Nothing else is allowed.

## Setup (about 10 minutes)

### 1. Run it on demand first

```bash
git clone https://github.com/pdichone/supabase-autopilot-starter.git
cd supabase-autopilot-starter
export SUPABASE_PROJECT_REF=your20charref     # from the project URL: dashboard/project/<ref>
claude
```

In Claude Code: `/mcp`, select `supabase-monitor`, **Authenticate**, approve in the browser. You should see seven tools: `execute_sql`, `get_advisors`, `list_extensions`, `list_migrations`, `list_tables`, `query_logs`, `search_docs`. No write tools; `read_only=true` hid them.

Paste the contents of `routine-prompt.md` (everything below the line). The run fetches Supabase's detection checks, queries the last two complete hours of API and Auth logs, reads connection pressure, and writes `state/health.json`. About two minutes.

Supabase's own guidance: run it on demand and read the first results before you schedule it.

### 2. Create a scoped token

Supabase dashboard, **Account** > **Access Tokens** (https://supabase.com/dashboard/account/tokens) > **Generate new token**.

- Name: `autopilot-health-ro`
- Scope: the one project you want to monitor
- Permissions: **Database: Read**, **Advisors: Read**, **Logs: Read**. Nothing else.

Copy the token (it starts with `sbp_fc`). It is shown once. It does not go in this repo or in any file.

### 3. Push the repo and create the cloud environment

Push this repo to your own GitHub account; the routine clones it each run and commits state back to `main`.

At claude.ai/code, open the environment selector and **Add cloud environment**:

- Name: `supabase-autopilot`
- **Network access**: **Custom**. Allowed domains: `supabase.com` and `*.supabase.com`. Tick **Also include default list of common package managers**. The prompt fetches `supabase.com/docs/...detecting.md` over plain HTTPS and the MCP server is at `mcp.supabase.com`; neither is on the default allowlist, and a repo `.mcp.json` server is not a connector, so its traffic goes through this allowlist.
- **Environment variables**, one line: `SUPABASE_PROJECT_REF=your20charref`
- **Network secrets** > **Add secret**: name `supabase-mcp-token`, credential type **Bearer**, allowed websites `mcp.supabase.com`, path prefixes empty, custom header `Authorization` with prefix `Bearer` and the token as the value. **Connect**.

Anthropic's agent proxy adds that header to every request the session makes to `mcp.supabase.com`, after the request leaves the sandbox. The session never sees the token.

### 4. Create the routine

At https://claude.ai/code/routines (or `/schedule` in the CLI):

- Instructions: paste `routine-prompt.md`, everything below the line
- Repository: this repo, and only this one (a routine with one repository loads the committed `.mcp.json`; with two it does not)
- Environment: `supabase-autopilot`
- Connectors: **remove every connector**. A routine can call every tool on an included connector, including writes, without asking.
- Trigger: **Custom**, cron `7 * * * *`. Runs scheduled exactly on the hour can start late.

Click **Run now**. Open the run session to read the findings block, and check `main` for the `chore(health): run ...` commit. The first run has no prior state and says so; from the second run on, unchanged findings are suppressed and resolved ones are marked so a recurrence notifies.

## What "read-only" means here, three layers

1. `read_only=true` on the MCP URL: SQL runs as the `supabase_read_only_user` role and mutating tools are excluded.
2. `features=docs,debugging,database`: no account, branching, functions or storage tools are registered at all.
3. The token has only Read permission on Database, Advisors and Logs. Even if the prompt asked for a write, the token can't do it.

## Alternative: token as an environment variable (Team and Enterprise, or self-hosted)

Network secrets are a Pro and Max feature. Without them, set `SUPABASE_ACCESS_TOKEN` as an environment variable on the cloud environment and add a header to `.mcp.json`:

```json
"headers": { "Authorization": "Bearer ${SUPABASE_ACCESS_TOKEN}" }
```

Claude Code expands `${VAR}` in `url` and `headers` at startup and disables the OAuth fallback once a header is set, so a missing variable fails fast with a 401 instead of opening a browser. Environment variables are readable by anyone who can use the environment. Locally, export both variables in the same terminal you start `claude` from and check with `claude mcp list` (it warns about missing variables).

## Swap in the other monitors

Create a second routine, paste `prompts/security.md` (daily), `prompts/performance.md` (hourly) or `prompts/usage.md` (daily) in place of the Health block in `routine-prompt.md`, and change `state/health.json` to `state/security.json` and so on in the harness lines.

## Two projects

Add a second server to `.mcp.json` (for example `supabase-monitor-staging`) with its own `project_ref` variable, add a second network secret for it (same host, so give the two tokens different path prefixes or use one token scoped to both projects), and tell the prompt which server to use for which project.

## Caveats, straight from Supabase's docs

- Logs and query results can contain secrets or personal data. The prompt aggregates and redacts; keep it that way.
- Every scheduled run starts with a fresh context. The prompt is self-contained on purpose.
- The first run has no prior state, so it reports that deduplication is unavailable. From the second run on, unchanged findings are suppressed.
- A green run status means the session started and exited. Open the run to see what Claude actually found.

## Sources

- Changelog: https://supabase.com/changelog/50403-observability-autopilot
- Agent prompts: https://supabase.com/docs/guides/observability/automate-with-agents
- Detection checks (what the prompt fetches): https://supabase.com/docs/guides/observability/detecting
- Supabase MCP server: https://supabase.com/docs/guides/getting-started/mcp
- Scoped personal access tokens: https://supabase.com/docs/guides/platform/personal-access-tokens
- Claude Code routines: https://code.claude.com/docs/en/routines
- Claude Code cloud environments: https://code.claude.com/docs/en/cloud-environments

MIT. Use it however you like.
