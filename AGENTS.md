You are a senior full-stack engineer and product designer working inside Antigravity. We are 3 students building "AutoHeal EvidenceFirst" in 6 hours for the CTRL+AI Challenge, problem statement PS-12 Auto-Heal. Build it as a Streamlit app (Python 3.11) deployed on Streamlit Community Cloud from a public GitHub repo. Work in Planning mode: write a short plan first, then implement in small commits, then run tests.

PRODUCT
An evidence-first incident-response agent for a SIMULATED orders platform with four services. The LLM (Gemini Flash-Lite via the google-genai SDK) investigates read-only first, keeps 2-3 competing root-cause hypotheses on a visible board, applies only a fix that the code-enforced safety gate allows, verifies recovery, learns from the incident, and escalates to a human with an evidence packet when it should not act. Demo scenarios: (1) cache outage, auto-fixed, (2) repeat cache outage, faster via incident memory, (3) bad config deployment, rollback needs human approval, (4) database integrity fault, repair is blocked, agent pages a human.

HARD RULES
- Safety is enforced in CODE, never only in the prompt. Unknown tools are blocked.
- Real reasoning, deterministic world: the simulator returns fixed results per fault; the LLM genuinely chooses tools. Do not script the order of the LLM's checks.
- Only app.py and ui/ import streamlit. Everything in core/ is plain Python operating on one dict S, so it is testable without a browser.
- All state lives in st.session_state["S"] (Streamlit reruns the script on every click).
- Never commit secrets. Read keys with st.secrets.get("GEMINI_API_KEY", ""). Empty key means fallback mode.
- No random numbers except +/-1% cosmetic noise on healthy metrics.
- Do not edit files owned by another member. If you need a change there, add a TODO note in docs/requests.md.
- Keep the model name in ONE constant MODEL_NAME at the top of core/agent.py.

REPO STRUCTURE
autoheal/
  app.py            (Member C)
  ui/               (Member C: theme.py, components.py, topology_svg.py)
  requirements.txt  .gitignore (.streamlit/secrets.toml, .env, __pycache__)
  .streamlit/config.toml  (Member C)
  core/
    state.py        init_state(), reset_environment(), PHASES (Member A)
    sandbox.py      services, dependency graph, metrics, logs (A)
    faults.py       inject_cache_outage(), inject_bad_config(), inject_db_integrity() (A)
    verifier.py     verify_recovery() (A)
    tools.py        tool functions + TOOL_DECLARATIONS for Gemini (B)
    guardrails.py   RISK table + execute_tool() (B)
    agent.py        run_agent(), resume_after_approval(), Gemini loop, system prompt (B)
    fallback.py     rule-based policy (B)
    memory.py       signature, save, find, postmortem (B)
    escalation.py   build_escalation_packet() (B)
  tests/            pytest tests for core/ (each owner tests own files)
  data/seed_incidents.json  (empty list)

SHARED STATE (one dict S)
phase: IDLE | INVESTIGATING | AWAITING_APPROVAL | REMEDIATING | VERIFYING | RESOLVED | ESCALATED
services: {orders_api, worker, cache, database} each with status and numbers
config: {current, previous}; deployments: list
incident: None or {id, scenario, started_at, alert, signature}
hypotheses: [{cause, confidence 0-1, status: open|eliminated|confirmed, evidence}]
timeline: [{t, actor: agent|human|system, kind, text, tier}]
contents: Gemini conversation history (list of types.Content), kept so the agent can pause and resume
pending: None or {tool, args, reason, risk}
memory: list of resolved incidents; llm_calls: int; mode: "gemini" | "fallback"; mode_reason: str
diag_calls: int; recovery_seconds: None or float

SIMULATOR
Baseline (all healthy): orders_api error 0.3%, latency 120 ms, memory 41%; worker queue 4; cache reachable, hit rate 94%; database 18/100 connections, integrity ok. Dependencies: orders_api -> cache, database, worker; worker -> cache.
Fault 1 cache outage: cache down/unreachable; worker degraded, queue 4 -> 380; orders_api down, error 48%, latency 2400 ms, memory ~43%; database healthy. Logs: ERROR cache unreachable: connection refused / WARN worker queue growing / ERROR orders_api 503 upstream timeout. Right fix: restart_cache (reversible, automatic), then verify.
Fault 2 bad config: deployment v2.4.1 went out 12 minutes ago; orders_api down, error 100%; others healthy; config diff shows DATABASE_URL removed. Logs: ERROR missing required env DATABASE_URL / FATAL orders_api failed to start. Right fix: rollback_config to v2.4.0 (needs human approval), then verify.
Fault 3 = fault 1 repeated, with memory match.
Fault 4 database integrity: database degraded, integrity check failed; orders_api intermittent errors 22%; cache and worker healthy. Logs: CRITICAL checksum mismatch on orders table / CRITICAL transaction integrity validation failed. No automatic fix: page_human with evidence packet.
restart_cache sets cache healthy and drains the queue over 2-3 simulated steps. rollback_config restores previous config and brings the API back. Nothing else repairs anything.
verify_recovery passes only if, for 3 checks in a row 1 second apart: orders_api ok, error rate < 2%, latency < 300 ms, worker queue < 20, cache reachable, database integrity ok.

TOOLS (name, tier, args)
Read tier: get_alert(), get_topology(), query_logs(service, minutes), get_metrics(service), check_health(service), ping_dependency(from_service, to_service), diff_config(service), get_recent_deployments(), find_similar_incident(signature), update_hypotheses(hypotheses list; screen only), verify_recovery(), page_human(summary, evidence, recommended_next_step), resolve_incident(summary; accepted only if the last verify_recovery passed).
Reversible: restart_cache().
Approval: rollback_config(service, to_version).
Blocked: repair_database() (expose it to the model on purpose so a blocked attempt is shown live as proof that code, not the prompt, enforces safety).
save_postmortem is NOT a model tool: code calls it automatically after verification passes.

SAFETY GATE (core/guardrails.py)
RISK = {read tools: 'read', 'restart_cache': 'reversible', 'rollback_config': 'approval', 'repair_database': 'blocked'}
execute_tool(name, args, approved=False): tier = RISK.get(name, 'blocked'); read/reversible run; approval returns {'status':'approval_required', ...} unless approved=True; blocked returns {'status':'blocked','message':'Blocked by safety policy.'}. Every call (allowed, paused, blocked) is appended to S['timeline'] with its tier.

GEMINI AGENT LOOP (core/agent.py)
- from google import genai; from google.genai import types; client = genai.Client(api_key=key).
- config = types.GenerateContentConfig(system_instruction=SYSTEM_PROMPT, tools=[types.Tool(function_declarations=TOOL_DECLARATIONS)], temperature=0.2, automatic_function_calling=types.AutomaticFunctionCallingConfig(disable=True), tool_config=types.ToolConfig(function_calling_config=types.FunctionCallingConfig(mode='ANY')), http_options with a 20 second timeout).
- Loop up to MAX_TURNS=12. If S['llm_calls'] >= CALL_LIMIT (30) switch to fallback with reason. Call generate_content; retry once on error or invalid output; on second failure switch to fallback and label why. Append the model's full response content to S['contents'] unchanged (this preserves any thought signatures).
- Take the FIRST function_call. For any extra function_calls in the same response, send function_response {'status':'skipped','message':'one tool at a time'}.
- Run execute_tool. Count diagnostic (non-screen, non-terminal) read calls in S['diag_calls']; at 5 the system must tell the model: decide now (fix, or page_human).
- If approval_required: store S['pending'], set phase AWAITING_APPROVAL, return (pause).
- Otherwise send a function_response part (role 'user') with the result and continue.
- Terminal: page_human sets phase ESCALATED; resolve_incident only counts if the last verify_recovery passed, then phase RESOLVED, stop the timer, auto-save the postmortem.
- Approve button: execute_tool(tool, args, approved=True), append the function_response for that call, phase REMEDIATING, call run_agent again so the model verifies. Reject button: append a function_response saying the human rejected, so the model calls page_human.
- Always give the model a function_response for every function_call it made.
- Timer starts when the incident is created and stops only after verification passes.

SYSTEM PROMPT
You are AutoHeal, an evidence-first incident-response agent for a simulated orders platform. Rules: 1) Begin with read-only checks; never choose a fix before you have evidence. 2) Keep 2 to 3 competing root-cause hypotheses on the board; call update_hypotheses after every tool result. 3) Mark a hypothesis eliminated only when a tool result contradicts it and confirmed only when a tool result supports it. 4) Call exactly one tool at a time and only from the provided tools. 5) Before a fix, state evidence, confidence, risk, rollback plan and verification checks. 6) After any fix, call verify_recovery; do not say recovered unless it passes, then call resolve_incident. 7) If no safe approved tool fits, confidence is below 0.6, risk is high, or verification fails, call page_human with your evidence. 8) Be concise.

FALLBACK (core/fallback.py): same tools, fixed order, same hypothesis board, runs when key missing, call failed, output invalid or call limit reached. 'cache unreachable' in logs -> get_metrics, check_health(database), ping_dependency, restart_cache, verify, resolve. 'DATABASE_URL' -> diff_config, get_recent_deployments, request rollback (pause), verify, resolve. 'checksum mismatch' -> check_health(database), page_human, no fix. Anything else -> page_human with a low-confidence note. UI must always show 'Agent mode: Reliable fallback policy' plus the reason when active.

MEMORY (core/memory.py)
signature built by code from the alert and a log scan, e.g. orders_api_errors|cache_unreachable|worker_queue_high. On a new incident the code calls find_similar_incident first; if found, the runbook is added to the first user message, but the agent must still run ONE confirmation check before acting. Save {id, signature, root_cause, runbook, steps_taken, recovery_seconds, postmortem} after verification. Compare first vs repeat: recovery seconds, steps, percent change. All real measured values. Keep in S['memory'].

PANELS: chaos controls (sidebar, behind CHAOS_ACCESS_CODE; label it a demo-control gate, not security), alert card, service topology, hypothesis board, timeline, approval card (Approve/Reject), escalation packet, memory comparison chart, agent mode label.

WORKING RULES
Plan first. Small commits. git pull --rebase before push. Write pytest tests for your own core files and run them. After each milestone post a 5-line summary: what works, what is stubbed, what is next. Do not add libraries beyond requirements.txt without asking. Prefer simple, readable code over clever code.
