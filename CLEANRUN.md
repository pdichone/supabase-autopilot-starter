# Clean-environment check

Run this in a fresh directory before recording, and again before publishing. All three must pass or the unit is not ready.

```bash
export SUPABASE_PROJECT_REF=your20charref     # not a secret; it is in your project URL

git clone https://github.com/pdichone/supabase-autopilot-starter.git fresh-check
cd fresh-check
claude mcp list        # must NOT print "Missing environment variables"
claude
```

1. `/mcp` lists `supabase-monitor`. Select it, **Authenticate**, approve in the browser. It then shows connected with exactly these seven tools: `execute_sql`, `get_advisors`, `list_extensions`, `list_migrations`, `list_tables`, `query_logs`, `search_docs`. No `apply_migration` (hidden by `read_only=true`) and no `get_logs`. Verified 2026-10-06 on Claude Code 2.1.292. No token is needed locally.
2. Paste `routine-prompt.md` (everything below the line). The run completes in about two minutes with a findings block that has a result line for both Health checks (`finding`, `clear`, or `unable to assess`), the connection snapshot reports the role `supabase_read_only_user`, and `state/health.json` is written and committed to `main`.
3. In the routine (claude.ai/code/routines), **Run now** produces the same findings block in the run session, and a new commit `chore(health): run ...` lands on `main` with no extra `claude/` branch.

If step 1 shows "Pending approval", run `claude` in the folder once and accept the workspace trust dialog, then re-run `/mcp`.
If step 3 reports "unable to assess" for everything, the run could not fetch `supabase.com`: add `supabase.com` and `*.supabase.com` to the environment's allowed domains.
If step 3 fails on `supabase-monitor` with a 401, the Network secret is missing or its allowed website is not `mcp.supabase.com`.
