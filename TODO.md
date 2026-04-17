
## Architecture: Claude-as-Planner + Local LLM-as-Orchestrator

**Core insight:** Claude gets censored on pentest payloads. Local models don't.
Better to use Claude only for what it's good at (strategy, reasoning) and local
models for everything sensitive (payload gen, exploitation, full orchestration).

### Two-Phase Architecture

**Phase 0 — Claude creates the roadmap (one call, no target data)**
- Input to Claude: ONLY technology stack fingerprint, e.g.:
  `{stack: "PHP 7.4 / MySQL 5.7 / nginx 1.18 / jQuery 3.2.1 / Laravel 8"}`
- Claude outputs a structured pentest roadmap JSON:
  `{phases: [...], priority_vulns: [...], suggested_payloads_by_type: [...], agent_sequence: [...]}`
- Claude never sees: domain, IP, company name, any response data

**Phase 1+ — Local LLM orchestrates everything**
- qwen3-coder-30b-a3b (MoE, ~50s/query) takes the roadmap as system context
- Acts as orchestrator: decides agent sequencing, interprets results, escalates findings
- Local agents generate real payloads, run exploitation — no censorship
- No anon wrapper needed: Claude data-blind by design

### Why this is better than anon-wrapper approach
- No stripping/tokenizing overhead
- No risk of accidental data leaks through anon wrapper bugs
- Claude used for exactly one call → minimal cost
- Local model unconstrained for payload generation
- Works fully offline after Phase 0

### Implementation sketch
```
shannon run <target> --planner claude --executor qwen3-coder-30b-a3b-instruct
```
1. Recon agent collects stack fingerprint (no LLM needed — whatweb/nmap output)
2. Single Claude API call: `planPentest(stackFingerprint)` → roadmap JSON
3. Roadmap injected as system prompt for local orchestrator
4. Local qwen3-coder-30B drives all remaining agents
5. Report generated locally

### Key files to modify
- `src/ai/claude-executor.js` — add `plannerMode` that returns after roadmap generation
- `src/cli/commands/RunCommand.js` — add `--planner` and `--executor` options
- New file: `src/ai/planner-client.js` — Claude-only planner (stack → roadmap)
- `prompts/planner-roadmap.txt` — prompt for Claude roadmap generation

