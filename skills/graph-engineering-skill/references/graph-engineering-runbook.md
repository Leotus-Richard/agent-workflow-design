# Graph Engineering Runbook: Converting a Linear Agent Workflow into an Execution Graph

Source framework: rari (@whrrari), [Graph Engineering: How to Build AI Agent Systems That Don't Break at Scale](https://whrrari.substack.com/p/graph-engineering-how-to-build-ai) (Aug 12, 2026), with layering context from [The Three Layers Behind Reliable AI Agents](https://whrrari.substack.com/p/the-three-layers-behind-reliable). Code patterns are TypeScript (Node 20+, `zod` for contracts). Patterns marked "(guide)" are adapted directly from the article; everything else is an implementation extension of its principles.

---

## 0. Vocabulary (guide)

| Primitive | Definition | Design question |
|---|---|---|
| NODE | One bounded unit of work (agent, tool call, deterministic function, verifier, human approval) | Can I define input, output, and failure independently? |
| EDGE | A real dependency: A produced data (or authority) B is allowed to consume | What data crosses this boundary? |
| STATE | Data that survives between nodes | Where can execution safely resume? |
| ROUTER | Rule that selects the next edge | Can I explain why this route was chosen? |
| GATE | Check that decides whether work may continue | On fail: repair, stop, or escalate? |

Layering: the harness is the environment, the loop is the feedback, the graph is the flow. The graph runs inside the harness; loops run inside individual nodes.

---

## 1. Reference Example: The Linear Workflow We Are Converting

A typical "research and publish" agent, written the way people write instructions:

```
1. Parse the request            6. Deduplicate sources
2. Classify complexity          7. Draft the article
3. Search company sources       8. Fact-check / cite
4. Search papers                9. Editorial review
5. Search expert posts         10. Human approval -> publish
```

Run as a single chain, steps 3-5 wait for each other, all ten steps share one bloating context window, and a failure at step 8 restarts everything.

---

## 2. Phase 1 — Dependency Audit (cut false edges)

The core test (guide): **Does the next step actually read the previous step's output? If no, cut the edge.**

### Procedure

1. List every step and every implicit "and then".
2. For each step, write down the exact fields it reads.
3. Draw an edge only from the producer of each field to its consumer.
4. Also keep edges that carry authority (e.g., approval required before publish) even if no data is read.
5. Anything with no incoming data edges from its siblings becomes a fan-out candidate.

### Audit worksheet

| Step | Reads (fields) | Produced by | Real edge? | Verdict |
|---|---|---|---|---|
| classify | `scope.question` | scope | yes | chain |
| search_company | `lanes[company]` | decompose | yes | fan-out branch |
| search_papers | `lanes[papers]` | decompose | yes | fan-out branch |
| search_papers ← search_company | nothing | — | **no** | cut edge |
| search_experts ← search_papers | nothing | — | **no** | cut edge |
| dedupe | all `findings[]` | all search branches | yes | **join (needs full set)** |
| draft | `evidence[]` | dedupe | yes | chain |
| verify | `draft`, `evidence[]` | draft, dedupe | yes | gate |
| publish | `draft` + `approval` | verify, human | yes (data + authority) | gated chain |

### Automating the audit

Declare reads/writes per node and derive edges, rather than hand-drawing arrows:

```ts
type NodeSpec = { id: string; reads: string[]; writes: string[] };

export function deriveEdges(nodes: NodeSpec[]) {
  const producer = new Map<string, string>();
  for (const n of nodes) for (const w of n.writes) {
    if (producer.has(w)) throw new Error(`Field ${w} written by two nodes; add a reducer`);
    producer.set(w, n.id);
  }
  const edges: [string, string, string][] = [];
  for (const n of nodes) for (const r of n.reads) {
    const p = producer.get(r);
    if (!p) throw new Error(`${n.id} reads ${r} but nothing produces it`);
    edges.push([p, n.id, r]); // [from, to, field]
  }
  return edges; // any pair not in here has NO dependency -> can run in parallel
}
```

Topological layers of the derived DAG give you the maximum safe parallelism for free.

---

## 3. Phase 2 — Node Contracts

Every node needs four things (guide): **one job, explicit input, structured output, a clear failure state.** A node you cannot describe precisely cannot be routed, tested, or replaced.

### Contract template (guide)

```json
{
  "node": "source_researcher",
  "input":  { "topic": "string", "source_type": "primary" },
  "output": { "claim": "string", "source_url": "string", "confidence": "high | medium | low" },
  "failure": "no_primary_source_found"
}
```

### Typed implementation

```ts
import { z } from "zod";

export type FailurePolicy = "RETRY" | "FALLBACK" | "SKIP" | "REPAIR" | "ESCALATE" | "STOP";

export type NodeResult<O> =
  | { ok: true; output: O }
  | { ok: false; failure: string; policy: FailurePolicy; detail?: unknown };

export interface NodeDef<I, O> {
  id: string;
  kind: "llm" | "tool" | "code" | "verifier" | "human";
  input: z.ZodType<I>;
  output: z.ZodType<O>;
  failures: Record<string, FailurePolicy>;   // enumerated, not free text
  maxAttempts?: number;
  timeoutMs?: number;
  optional?: boolean;                         // SKIP-able branch
  run(input: I, ctx: RunContext): Promise<O>;
}

export const SourceResearcher: NodeDef<
  { topic: string; source_type: "primary" | "secondary" },
  { claims: { claim: string; source_url: string; confidence: "high" | "medium" | "low" }[] }
> = {
  id: "source_researcher",
  kind: "llm",
  input: z.object({ topic: z.string(), source_type: z.enum(["primary", "secondary"]) }),
  output: z.object({
    claims: z.array(z.object({
      claim: z.string(),
      source_url: z.string().url(),
      confidence: z.enum(["high", "medium", "low"]),
    })).min(1),
  }),
  failures: {
    no_primary_source_found: "SKIP",
    tool_timeout: "RETRY",
    schema_invalid: "REPAIR",
  },
  maxAttempts: 3,
  timeoutMs: 60_000,
  optional: true,
  async run(input, ctx) {
    return ctx.llm.structured({ model: "small", schema: this.output, prompt: `Find primary claims on ${input.topic}` });
  },
};
```

### Contract wrapper (enforces schemas at both boundaries)

```ts
export async function execNode<I, O>(def: NodeDef<I, O>, raw: unknown, ctx: RunContext): Promise<NodeResult<O>> {
  const parsedIn = def.input.safeParse(raw);
  if (!parsedIn.success) return { ok: false, failure: "input_invalid", policy: "STOP", detail: parsedIn.error };
  try {
    const out = await withTimeout(def.run(parsedIn.data, ctx), def.timeoutMs ?? 120_000);
    const parsedOut = def.output.safeParse(out);
    if (!parsedOut.success)
      return { ok: false, failure: "schema_invalid", policy: def.failures.schema_invalid ?? "REPAIR", detail: parsedOut.error };
    return { ok: true, output: parsedOut.data };
  } catch (e: any) {
    const failure = e?.code ?? "unknown_error";
    return { ok: false, failure, policy: def.failures[failure] ?? "ESCALATE", detail: String(e) };
  }
}
```

Rule: swapping the model, prompt, or tool is safe as long as the contract stays the same.

### Node-kind selection

| Work type | Node kind | Model tier |
|---|---|---|
| Flatten, dedupe, filter, join, status check | `code` | none |
| Extraction, classification, formatting | `llm` | small/cheap |
| Decomposition, synthesis, hard verification | `llm` | strong |
| Tests, schema, citation-URL resolution | `verifier` (code) | none |
| Irreversible or high-risk actions | `human` | — |

---

## 4. Phase 3 — Edges as Data Contracts

An edge means "A produced data that B is allowed to consume", not "B comes after A" (guide). Plumbing is code, not model calls: "A graph where every edge is another agent is paying tokens for its own wiring."

```ts
// Deterministic join/normalize (guide pattern)
export function dedupeFindings(results: (Findings | null)[]) {
  const usable = results.filter(Boolean).flatMap(r => r!.claims);
  return [...new Map(usable.map(c => [normalizeUrl(c.source_url), c])).values()];
}

export interface Edge {
  from: string;
  to: string;
  carries: string[];                         // state fields consumed by `to`
  select?: (s: GraphState) => unknown;       // projection: pass only what is read
}
```

Pass projections or artifact references, never whole transcripts.

---

## 5. Phase 4 — The Four Shapes (guide)

```
CHAIN            A -> B -> C                       every step needs the previous output
DIAMOND          A -> {B1,B2,B3} -> C              independent branches, deliberate join
ROUTER           CLASSIFY -> QUICK | FULL AUDIT    pay only for the path the task needs
CONTROLLED CYCLE WORK -> VERIFY -> PASS -> EXIT    repeat only on evidence, with hard stops
                             \-> FAIL -> FEEDBACK -> WORK
```

Selection rule: real dependency → chain; independent work → diamond; risk-dependent effort → router; verify-and-repair → controlled cycle. Combine as needed.

---

## 6. Phase 5 — Fan-Out, Joins, and Streaming

### Fault-isolated fan-out (guide)

```ts
export async function fanOut<T, O>(items: T[], node: (t: T) => Promise<O>, concurrency = 8) {
  const limit = pLimit(concurrency);                    // bound concurrency (rate limits, cost)
  const settled = await Promise.allSettled(items.map(i => limit(() => node(i))));
  const ok = settled.filter((r): r is PromiseFulfilledResult<O> => r.status === "fulfilled").map(r => r.value);
  const failed = settled.flatMap((r, i) => r.status === "rejected" ? [{ item: items[i], reason: r.reason }] : []);
  return { ok, failed };                                // one failed branch never kills the others
}
```

### Quorum join (avoid waiting on stragglers)

```ts
export async function joinQuorum<O>(tasks: Promise<O>[], { min, deadlineMs }: { min: number; deadlineMs: number }) {
  const done: O[] = [];
  await Promise.race([
    new Promise<void>(res => tasks.forEach(t => t.then(v => { done.push(v); if (done.length >= min) res(); }).catch(() => {}))),
    sleep(deadlineMs),
  ]);
  if (done.length < min) throw Object.assign(new Error("quorum_not_met"), { code: "quorum_not_met" });
  return done;   // late results can be appended to state asynchronously
}
```

### Streaming vs barrier

Place a join only when the next node needs the complete set (guide): cross-source dedupe, ranking all candidates, comparing alternatives, coverage decisions. Otherwise stream per item:

```ts
// Each item continues independently — no barrier between stages
export async function streamPipeline<T>(items: AsyncIterable<T>, stages: ((x: any) => Promise<any>)[]) {
  const inflight: Promise<any>[] = [];
  for await (const item of items) {
    inflight.push(stages.reduce((p, stage) => p.then(stage), Promise.resolve(item)));
  }
  return Promise.allSettled(inflight);
}
```

"Parallel is not automatically fast. Your topology decides where the system waits." (guide)

### Latency checklist

- Critical path = longest chain of real edges; optimize only nodes on it.
- Bound concurrency per provider; use per-branch timeouts.
- Prefer quorum/deadline joins over all-or-nothing barriers when completeness is not required.
- Speculatively start likely-needed branches only when they are cheap and side-effect free.

---

## 7. Phase 6 — Inspectable Routing

The classifier is probabilistic; the allowed routes are deterministic (guide).

```ts
const RiskDecision = z.object({
  severity: z.enum(["low", "high", "unknown"]),
  rationale: z.string(),
  confidence: z.number().min(0).max(1),
});

export async function routeChange(change: Change, ctx: RunContext) {
  const d = RiskDecision.parse(await ctx.llm.structured({ model: "small", schema: RiskDecision, prompt: classifyPrompt(change) }));
  const severity = d.confidence < 0.7 ? "unknown" : d.severity;           // low confidence never takes the cheap path
  const route = ({ low: "quick_review", high: "full_parallel_audit", unknown: "human_review" } as const)[severity];
  await ctx.state.recordDecision({ node: "risk_router", route, inputs: { changeId: change.id }, decision: d });
  return route;   // only members of this closed set are reachable
}
```

Rules:
- Routes are a closed enum declared in the graph; the model picks, code enforces.
- Default/unknown routes go to the safer path, never the cheaper one.
- Every decision is logged with the state that produced it.

---

## 8. Phase 7 — Verification Gates on Edges

The highest-leverage node often produces nothing new; it stops weak work moving downstream (guide). Never let one agent generate, approve, and publish in one context.

```ts
type GateCheck = { name: string; pass: boolean; section?: string; detail?: string };

export async function verifyDraft(draft: Draft, evidence: Evidence[], ctx: RunContext) {
  const checks: GateCheck[] = [
    ...draft.claims.map(c => ({ name: "claim_has_source", pass: !!c.sourceId, section: c.section })),
    ...await Promise.all(draft.claims.map(async c => ({
      name: "source_supports_claim",
      pass: await ctx.llm.judge({ model: "strong", claim: c.text, source: evidence.find(e => e.id === c.sourceId) }),
      section: c.section,
    }))),
    { name: "schema", pass: DraftSchema.safeParse(draft).success },
    { name: "required_sections", pass: REQUIRED.every(s => draft.sections.has(s)) },
  ];
  const failedSections = [...new Set(checks.filter(c => !c.pass).map(c => c.section).filter(Boolean))];
  return { pass: checks.every(c => c.pass), checks, failedSections };   // REPAIR only the failing sections
}
```

Gate types: deterministic (schema, tests, URL resolves), independent-model judgment (separate prompt/context), cross-path agreement, and human approval for irreversible actions.

---

## 9. Phase 8 — Durable State and Checkpoints

### State schema (guide fields)

```ts
export const GraphState = z.object({
  task_id: z.string(),
  current_node: z.string().nullable(),
  completed_nodes: z.array(z.string()),
  artifacts: z.record(z.object({ uri: z.string(), sha256: z.string(), producedBy: z.string() })), // references, not transcripts
  decisions: z.array(z.object({ node: z.string(), route: z.string(), at: z.string(), inputs: z.unknown(), decision: z.unknown() })),
  evidence: z.array(z.object({ id: z.string(), url: z.string(), claim: z.string() })),
  budgets: z.object({ tokens_remaining: z.number(), usd_remaining: z.number(), deadline: z.string() }),
  retry_counts: z.record(z.number()),
  human_approvals: z.record(z.object({ approver: z.string(), at: z.string(), artifactSha: z.string() })),
});
export type GraphState = z.infer<typeof GraphState>;
```

### Reducers for parallel writes

```ts
export const reducers: Partial<Record<keyof GraphState, (a: any, b: any) => any>> = {
  evidence: (a, b) => [...new Map([...a, ...b].map(e => [e.url, e])).values()],
  decisions: (a, b) => [...a, ...b],
  completed_nodes: (a, b) => [...new Set([...a, ...b])],
};
```

### Checkpointer

```ts
export interface Checkpointer {
  save(taskId: string, state: GraphState, version: number): Promise<void>; // optimistic concurrency
  load(taskId: string): Promise<{ state: GraphState; version: number } | null>;
}
// Checkpoint after every expensive node and before every side effect.
```

The graph must always answer: What has already happened? Why did it choose this route? Where can it safely resume? If not, it is still a demo (guide).

---

## 10. Phase 9 — Controlled Cycles

"Repeat until good" is not a stop condition (guide). Every cycle needs a completion test, max rounds, a cost budget, a record of prior attempts, and an escalation path.

```ts
export async function discoverUntilConverged(ctx: RunContext, opts = { maxRounds: 6, dryLimit: 2 }) {
  const seen = new Set<string>(ctx.state.get().evidence.map(e => e.url));   // includes rejected items
  let dry = 0, round = 0;
  while (dry < opts.dryLimit && round < opts.maxRounds) {
    if (ctx.budget.exhausted()) return { status: "STOP", reason: "budget" };
    const findings = await discover(ctx);
    const fresh = findings.filter(f => !seen.has(f.key));
    fresh.forEach(f => seen.add(f.key));                 // dedupe against everything seen, not only passed items
    dry = fresh.length === 0 ? dry + 1 : 0;
    round++;
    await ctx.checkpoint();
  }
  return round >= opts.maxRounds && dry < opts.dryLimit
    ? { status: "ESCALATE", reason: "did_not_converge" }
    : { status: "CONVERGED", rounds: round };
}
```

---

## 11. Phase 10 — Local Failure Policy

| Policy | Trigger (guide) | Implementation |
|---|---|---|
| RETRY | Transient tool/network failure | Exponential backoff + jitter, `maxAttempts` |
| FALLBACK | Preferred model/source unavailable | Alternate node with the same contract |
| SKIP | Optional branch failed | Mark branch skipped; join proceeds if quorum holds |
| REPAIR | Output failed validation | Route only the failing section/item to a repair node |
| ESCALATE | Risk/uncertainty over threshold | Human interrupt; persist state and pause |
| STOP | Budget, safety, or permission boundary | Terminate run, keep checkpoint |

```ts
export async function applyPolicy(r: NodeResult<any>, def: NodeDef<any, any>, raw: unknown, ctx: RunContext): Promise<NodeResult<any>> {
  if (r.ok) return r;
  const attempts = ctx.state.incRetry(def.id);
  switch (r.policy) {
    case "RETRY":
      if (attempts < (def.maxAttempts ?? 3)) { await sleep(backoff(attempts)); return applyPolicy(await execNode(def, raw, ctx), def, raw, ctx); }
      return { ...r, policy: "ESCALATE" };
    case "FALLBACK": return execNode(ctx.registry.fallbackFor(def.id), raw, ctx);
    case "SKIP":     return def.optional ? { ok: true, output: null } : { ...r, policy: "ESCALATE" };
    default:         return r;   // REPAIR / ESCALATE / STOP handled by the router
  }
}

// Idempotent side effects: retries never duplicate writes
export async function publishOnce(ctx: RunContext, artifactSha: string, publish: () => Promise<string>) {
  const key = `${ctx.state.get().task_id}:publish:${artifactSha}`;
  return ctx.kv.getOrSet(key, publish);
}
```

Also (guide): give parallel workers isolated workspaces (e.g., git worktrees) when they modify files.

---

## 12. Phase 11 — Topology as Cost Model

```
SIMPLE REQUEST  -> SMALL MODEL -> QUICK CHECK -> DONE
COMPLEX REQUEST -> PLANNER -> PARALLEL SPECIALISTS -> VERIFIERS -> STRONG SYNTHESIZER -> HUMAN GATE
```

```ts
export const modelFor = (kind: string) =>
  ({ extract: "small", classify: "small", format: "small", decompose: "strong", synthesize: "strong", verify_hard: "strong" } as const)[kind] ?? "small";

export function admitFanOut(lanes: number, ctx: RunContext) {
  const est = lanes * ctx.costModel.perLaneUsd;
  if (est > ctx.state.get().budgets.usd_remaining) return Math.floor(ctx.state.get().budgets.usd_remaining / ctx.costModel.perLaneUsd);
  return lanes;   // the graph, not the model, caps fleet size
}
```

Spend coordination only where parallelism, specialization, or independent verification earns it (guide).

---

## 13. The Converted Architecture (reference graph)

```
                       -> COMPANY_SOURCES --
TOPIC -> SCOPE -> ROUTE -> DECOMPOSE -> PAPERS ----------> DEDUPE (code, join)
          |                -> EXPERT_POSTS -----            |
          |                                                 v
          +--(simple)--> SMALL_ANSWER -> QUICK_CHECK -> DONE   DRAFT (strong)
                                                            |
                                     REPAIR(section) <-FAIL- FINAL_CHECK (verifier gate)
                                          |                 | PASS
                                          +----> DRAFT      v
                                                       HUMAN_GATE -> PUBLISH (idempotent)
```

Flow (guide): scope defines question, audience, completion criteria → decomposition creates independent lanes → research runs in parallel with separate contexts → code dedupes and normalizes → draft writes from structured evidence → checker validates claims, citations, style, sections → failures route only the relevant section to repair → human approves before publish.

### Declarative graph definition

```ts
export const graph = defineGraph({
  start: "scope",
  nodes: { scope, route, small_answer, quick_check, decompose, company_sources, papers, expert_posts,
           dedupe, draft, final_check, repair, human_gate, publish },
  edges: [
    { from: "scope", to: "route", carries: ["scope"] },
    { from: "route", router: s => s.route, to: { simple: "small_answer", complex: "decompose" } },
    { from: "small_answer", to: "quick_check", carries: ["answer"] },
    { from: "decompose", fanOut: ["company_sources", "papers", "expert_posts"], carries: ["lanes"] },
    { join: ["company_sources", "papers", "expert_posts"], to: "dedupe", mode: { quorum: 2, deadlineMs: 90_000 } },
    { from: "dedupe", to: "draft", carries: ["evidence"] },
    { from: "draft", to: "final_check", carries: ["draft", "evidence"] },
    { from: "final_check", router: s => s.check.pass ? "pass" : "fail", to: { pass: "human_gate", fail: "repair" } },
    { from: "repair", to: "final_check", cycle: { maxRounds: 3, onExhaust: "human_gate" } },
    { from: "human_gate", to: "publish", requires: "human_approvals.publish" },   // authority edge
  ],
});
```

### Minimal executor

```ts
export async function runGraph(g: Graph, taskId: string, ctx: RunContext) {
  let { state, version } = (await ctx.checkpointer.load(taskId)) ?? { state: g.initialState(taskId), version: 0 };
  let frontier = g.resumeFrontier(state);                       // nodes whose inputs are satisfied and not completed
  while (frontier.length) {
    const results = await Promise.allSettled(frontier.map(id => runWithPolicy(g.nodes[id], g.inputFor(id, state), ctx)));
    for (const [i, r] of results.entries()) {
      const id = frontier[i];
      if (r.status === "fulfilled" && r.value.ok) state = g.merge(state, id, r.value.output, reducers);
      else state = g.handleFailure(state, id, r);                // REPAIR / ESCALATE / STOP routing
      if (g.isInterrupt(state)) { await ctx.checkpointer.save(taskId, state, ++version); return { status: "PAUSED", state }; }
    }
    await ctx.checkpointer.save(taskId, state, ++version);
    frontier = g.nextFrontier(state);                            // evaluates routers, joins, cycle limits
  }
  return { status: "DONE", state };
}
```

---

## 14. Migration Steps (linear → graph)

1. Start with one loop. Instrument the existing chain: per-step latency, tokens, failure rate.
2. Run the dependency audit (Section 2); cut every edge that carries no data or authority.
3. Write contracts for every remaining step; replace free-text handoffs with schemas.
4. Move plumbing (dedupe, filter, join) from model calls to code.
5. Introduce the router so simple requests take the short path.
6. Convert independent steps to fan-out; place joins only where the full set is required.
7. Insert verifier gates before anything downstream trusts the output; separate generator and verifier contexts.
8. Add durable state, reducers, and checkpoints after expensive nodes.
9. Bound every cycle (rounds, budget, convergence, escalation).
10. Assign failure policies per node; make side effects idempotent.
11. Re-measure critical path, cost per request, and resume success; delete nodes that do not pay for themselves.

### When not to use a graph (guide)

Stay with one agent in one loop when the task is short, fits in one context, has no independent branches, failure is cheap, and a human can review the result quickly. Move to a graph when work parallelizes, nodes need different tools or permissions, outputs need independent verification, the task must resume after interruption, several loops share state, or cost and authority must be controlled by route.

---

## 15. Testing and Observability

- Contract tests: fuzz each node's input schema; assert outputs parse.
- Edge tests: `deriveEdges` must produce no orphan reads and no double writers.
- Routing tests: table-driven cases per route, including low-confidence → safe path.
- Chaos tests: inject failures per node; assert other branches complete and policy matches.
- Resume tests: kill the process after each checkpoint; assert identical final state and no duplicate side effects.
- Trace per node: `task_id, node, attempt, model, tokens, usd, latency_ms, result, route_chosen`.

---

## 16. Pre-Ship Checklist (guide)

```
[ ] Does every edge carry real data or authority
[ ] Does every node have one bounded job
[ ] Are inputs and outputs structured
[ ] Can independent nodes run in parallel
[ ] Are joins placed only where the full set is required
[ ] Are important results verified before moving downstream
[ ] Can failures be retried without duplicating side effects
[ ] Can the graph resume from a checkpoint
[ ] Does every cycle have a hard stop and budget
[ ] Can a human interrupt high-risk paths
[ ] Can you explain why every route was selected
[ ] Is the graph simpler than the problem it solves
```

If the last answer is no, delete nodes.

---

Sources: [whrrari — Graph Engineering guide](https://whrrari.substack.com/p/graph-engineering-how-to-build-ai); [whrrari — The Three Layers Behind Reliable AI Agents](https://whrrari.substack.com/p/the-three-layers-behind-reliable).
