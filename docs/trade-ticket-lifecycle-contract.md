# Trade Ticket Lifecycle Contract

> **Status:** V2 frontend read model, backend-ready `trade_ticket.v1` contract.

Diese Datei ist die **maßgebliche Grundlage** für alle Ticket-orientierten Frontend-Screens
(u. a. `frontend-plan/04-candidates.md`). Sie beschreibt das Read-Model und die harten Regeln.
Der Contract darf verbessert werden — Änderungen hier sind quellführend für die Screens.

## Purpose

This contract defines RickBrain's ticket-first Control Center. The first operator surface is not a
dashboard and not a signal bot. It is a **Ticket Control Room** that answers:

- Which trading idea is alive?
- In which lifecycle phase is it?
- Why may it continue?
- What would kill it?
- Does human governance need to inspect it?

Until a backend `trade_ticket.v1` endpoint exists, the frontend may project trade tickets from the
existing `CockpitSnapshot`. That projection is **read-only**. It must not recompute risk, create a new
trading decision, reinterpret market data, or add mutating trade controls.

## Identity

- **Trade Ticket** = fachliche Handelsakte for one trading thesis.
- **MT5-Ticket** = execution receipt from the terminal/bridge.
- `thesis_id` is the preferred business key. If an active source has no `thesis_id`, the frontend may
  create a synthetic inspection id such as `synthetic-BTCUSD`. A synthetic ticket is **never execution
  authority** and must never be used as proof that a trade is allowed.
- For active managed positions, the frontend may merge MT5-ticket records into the current `thesis_id`
  when the live position ticket matches the managed position. Closed historical MT5 tickets stay
  separate unless a backend trade-ticket projection explicitly links them.

## Lifecycle Phases

| Phase | Meaning | Human gate |
| --- | --- | --- |
| `asleep` / `no_ticket` | No active trade ticket exists for the asset. `no_ticket` is the frontend enum value. | No |
| `watch` | Radar/watchlist item exists, no candidate decision yet. | No |
| `gatekeeper_blocked` | Gatekeeper prevents scout/candidate work. | No by default |
| `scout_rejected` | Scout council rejected the idea before risk preflight. | No by default |
| `candidate` | Core candidate or entry candidate exists in Radar/Agentenrat. | No |
| `evidence_pending` | Required source, confirm/veto or council evidence is not complete yet. | Yes when governance cannot resolve policy automatically |
| `evidence_blocked` | Evidence, source health, policy or council result blocks before risk. | Yes when surfaced as exception |
| `risk_warn` | Risk preflight produced warning evidence. | Yes |
| `risk_blocked` | Risk preflight blocked/rejected the setup. | Yes |
| `exception` | Data, unmanaged position, blocker or inconsistent trace requires operator review. | Yes |
| `execution_pending` | An execution intent/order event exists but position state is not final in the read model. | No |
| `execution_rejected` | Execution path rejected after intent creation or order ingress. | Yes |
| `position_hold` | Managed position is open and no tighten/exit state is active. | No |
| `position_tighten` | Manager evidence requests tighter protection, reduce, trail or state-0 handling. | No by default; operator may review |
| `suspended` | System, symbol or autonomy state suspends continuation. | Yes |
| `exited` | Position or ticket path is closed. | No |
| `review` | Closed, rejected or special path needs post-trade review. | Yes |
| `lab_candidate` | Testcenter/lab-only candidate, never live authority. | Yes |
| `archived` | Historical ticket retained for audit/search. | No |

**Harder gates win** when multiple sources point to one ticket:

`risk_blocked` > `execution_rejected` > `exception` > `position_tighten` > `position_hold` > `candidate` > `watch`.

System suspension may move living tickets to `suspended`. Closed review/archive states should remain
inspectable instead of being rewritten by transient runtime health.

## TradeTicket Shape

The frontend `TradeTicket` read model is a **file, not just a row**:

- `identity`: `ticketId`, `thesisId`, `signalId`, `correlationId`, `mt5Ticket`, `symbol`, `synthetic`, `origin`.
- `thesis`: `playbook`, `actionIntent`, `statusReason`, `timeframe`, `direction`.
- `currentPhase`: phase, label, priority and `requiresHuman`.
- `continuationReason`: why this ticket is still allowed to live.
- `deathCondition`: what would stop or archive this ticket.
- `gateStack`: Radar, Gatekeeper, Scout, Evidence, Risk, Autonomy, Execution, Manager and Review gates.
- `council`: Agentenrat mode `scout`, `manager` or `review`, verdict, role summaries and reason codes.
- `riskPreflight`: read-only state and reason codes from existing risk evidence.
- `executionIntent`: existing execution refs only; no resend authority.
- `position`: existing position refs only.
- `exit`, `review`, `lab`: post-trade and testcenter/lab evidence.
- `exceptions[]`: governance inbox items.
- `refs[]`: audit/navigation references.
- `missingEvidence[]`: explicit list of absent data such as `thesis_id`, `decision_chain`,
  `risk_preflight` or `execution_intent`.

Missing data must be represented as `missing`, `not_available` or an empty array. The frontend must
**not invent a gate result**.

## Hard Gates

1. Risk preflight is authoritative for risk state. The UI only displays `risk_warn`, `risk_blocked`,
   reason codes and references that already exist in the snapshot.
2. A `risk_blocked` ticket must not show or trigger an execution intent action. Blocked risk means no
   downstream order authority.
3. Agentenrat output is evidence and audit context. It can support, warn, abstain or block, but it does
   not override risk preflight, suspend state, execution rejection or operator locks.
4. `execution_pending` is only a projection of an existing execution intent/order event. It is not
   permission to resend.
5. MT5-Ticket references are execution receipts. They are never the primary business identity of the
   trade ticket.
6. Missing `thesis_id` creates a visible synthetic inspection file only. It must remain read-only until
   backend identity is explicit.
7. Unmanaged/manual positions must surface as `exception` when they enter the ticket board.
8. The Ticket Control Room must not add buttons for buy, sell, order, deploy, start paper, suspend or
   stop mutation. **Navigation to existing guarded surfaces is allowed.**

## Agentenrat Boundary

The Agentenrat **contributes**:

- `decision_chain` stages and reason codes.
- summarized agent votes, abstentions and blocker evidence.
- `action_intent`, `playbook`, `thesis_stage`, source health and audit references.
- council mode for scout, manager and review presentation.

The Agentenrat **never contributes**:

- a new risk calculation in the frontend.
- permission to bypass risk preflight.
- permission to override execution rejection.
- raw private model thinking.
- direct trade mutation controls in the ticket board.

## Exception Inbox

The Exception Inbox separates human work from trade authority. It surfaces governance cases only:

- risk warning without clear policy.
- config mismatch or effective-config uncertainty.
- MT5/Box degraded or suspended.
- evidence source down.
- Agentenrat disagreement, abstention or incomplete confirm/veto evidence.
- repeated execution rejected.
- unexpected/manual/unmanaged position.
- review required after closed, rejected or incident paths.

The inbox may navigate to Operations, Agentenrat, Risk, Config or Testcenter. It must **not** offer
"take trade anyway" controls.

## Review And Lab

Closed or blocked tickets may enter review:

- **Light Review:** normal closed/rejected file inspection.
- **Full Review:** repeated blocker pattern or degraded evidence path.
- **Incident Review:** risk block, execution reject, manual/unmanaged position or system safety issue.

Lab candidates are visible patterns for safe Testcenter work: evidence blockers, risk blocks, execution
rejects, manager tighten cases and review learnings. A lab candidate is **never live trading authority**.

## Projection Rules

The frontend read model may build `TradeTicket` objects from:

- Market Radar active asset and watchlist entries.
- Agentenrat `thesis_id`, `decision_chain`, votes and source health.
- Demo/Paper lifecycle events and execution refs.
- MT5/Paper position refs.
- AgentTraceJournal records.

Projection may merge duplicate evidence into one trade ticket when the identity is stable. The UI should
prefer the highest-severity living phase and keep execution refs, position refs, review refs and
decision-chain stages deduped.

No frontend rule may reinterpret market data into a new trade decision. When data conflicts, the ticket
should become `exception`, keep the backend-provided status, or show `missingEvidence` instead of guessing.
