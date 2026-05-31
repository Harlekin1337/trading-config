# Kandidatenliste — `candidate`-Phase im Ticket Control Room

> Handelsideen, die *jetzt* die Setup-Schwelle erreicht haben und auf dem Weg durch die Gates sind —
> als **read-only Ticket-Akten** dargestellt, nicht als Order-Maske.

---

## 0. Leitprinzip (aus dem Contract)

Dieser Screen ist **kein** Order-Eingabeformular. Er ist ein **Ausschnitt des Ticket Control Room**
(`trade_ticket.v1`) und beantwortet für jede lebende Idee die fünf Leitfragen:

1. Welche Idee lebt?
2. In welcher Lifecycle-Phase ist sie?
3. **Warum** darf sie weiterleben? (`continuationReason`)
4. **Was würde sie töten?** (`deathCondition`)
5. Muss ein Mensch sie prüfen? (`requiresHuman`)

> **Read-only.** Die UI rechnet kein Risiko neu, trifft keine Handelsentscheidung, interpretiert keine
> Marktdaten neu und enthält **keine** Buttons für buy / sell / order / deploy / start paper / suspend /
> stop. Erlaubt ist nur: **inspizieren**, zu **abgesicherten Flächen navigieren**, in die **Exception
> Inbox / Review** geben.

---

## 1. Stellung im Lifecycle

```
   watch ──▶ candidate ──▶ evidence_* ──▶ risk_* ──▶ execution_* ──▶ position_* ──▶ exited/review
              ▲                                                                         
              │  Scout-Council (Agentenrat, mode = scout)                               
   gatekeeper_blocked / scout_rejected zweigen vorher ab                                
```

Dieser Screen zeigt das **Vorfeld der Ausführung** — Tickets, deren höchste lebende Phase eine der
folgenden ist (Auswahl über Phasen-Filter, Standard = die ersten vier):

| Phase | Zeigt der Screen | Standard sichtbar |
|-------|------------------|-------------------|
| `candidate` | Kern-/Entry-Kandidat existiert | ✅ |
| `evidence_pending` | Quellen/Confirm-Veto/Council unvollständig | ✅ |
| `evidence_blocked` | Evidence/Source/Policy/Council blockt vor Risk | ✅ |
| `risk_warn` | Risk-Preflight mit Warnung | ✅ |
| `risk_blocked` | Risk-Preflight blockt/rejected | optional (Filter) |
| `scout_rejected` | Scout-Council hat vor Risk abgelehnt | optional (Filter) |
| `gatekeeper_blocked` | Gatekeeper verhindert Scout-/Kandidatenarbeit | optional (Filter) |

> **Hardest gate wins.** Zeigen mehrere Quellen auf ein Ticket, gilt die Priorität aus dem Contract:
> `risk_blocked` > `execution_rejected` > `exception` > `position_tighten` > `position_hold` >
> `candidate` > `watch`. Der Screen führt jedes Ticket **genau einmal** in seiner höchsten lebenden Phase.

### Abgrenzung zur Watchlist

| | Watchlist (`watch`) | Kandidaten (`candidate` + Vorfeld) |
|---|-----------|------------|
| Ticket-Phase | `watch` | `candidate`, `evidence_*`, `risk_*`, `scout_rejected`, `gatekeeper_blocked` |
| Frage | „Was beobachten wir?" | „Welche Idee ist *jetzt* aktiv und warum darf sie weiter?" |
| Stabilität | stabil | dynamisch, flüchtig |
| Nutzeraktion | kuratieren | **inspizieren** + ggf. zur abgesicherten Fläche navigieren |
| Risiko | kein Risk-Preflight | Risk-Preflight-Status sichtbar (read-only) |

---

## 2. Datenquelle — `TradeTicket` Read-Model

Jede Zeile ist eine **Ticket-Akte** (`TradeTicket`), projiziert aus dem `CockpitSnapshot`
(Market Radar, Agentenrat, Demo/Paper-Events, Positions-Refs, AgentTraceJournal). Solange kein
Backend-`trade_ticket.v1`-Endpoint existiert, ist die Projektion **read-only**.

| Read-Model-Feld | Verwendung im Screen |
|-----------------|----------------------|
| `identity.symbol`, `thesisId`, `synthetic`, `origin`, `mt5Ticket` | Zeilen-Identität; Synthetic-Badge |
| `thesis.playbook`, `actionIntent`, `direction`, `timeframe`, `statusReason` | Setup-/Playbook-Spalte, Richtung |
| `currentPhase.{phase,label,priority,requiresHuman}` | Phasen-Pill, Sortierung, Governance-Marker |
| `continuationReason` | Spalte/Drawer: *warum lebt das Ticket?* |
| `deathCondition` | Spalte/Drawer: *was würde es töten?* |
| `gateStack` (Radar, Gatekeeper, Scout, Evidence, Risk, Autonomy, Execution, Manager, Review) | Gate-Stack-Anzeige im Drawer |
| `council` (mode=`scout`, verdict, role summaries, reason codes) | Scout-Council-Evidenz |
| `riskPreflight` (state + reason codes + refs) | **read-only** Risk-Status (nicht neu berechnet) |
| `missingEvidence[]` | explizite Lücken (z. B. `thesis_id`, `decision_chain`, `risk_preflight`) |
| `refs[]` | Navigation zu Operations / Agentenrat / Risk / Config / Testcenter |

> **Synthetic Tickets:** Fehlt `thesis_id`, entsteht eine sichtbare Inspektions-Akte `synthetic-<SYMBOL>`.
> Sie ist **nie** Ausführungs-Autorität und bleibt read-only, bis Backend-Identität vorliegt → deutliche
> `Synthetic`-Markierung.
>
> **Keine erfundenen Werte:** Fehlende Daten erscheinen als `missing` / `nicht verfügbar` / leere Liste —
> die UI errechnet **kein** Gate-Ergebnis und **keine** Positionsgröße.

---

## 3. Layout

```
┌────────────────────────────────────────────────────────────────────────────┐
│  ⓘ Autonomy: Risk-Off — Gatekeeper rät von neuen Einstiegen ab (Hinweis)    │  ← Info, blockt nicht
├────────────────────────────────────────────────────────────────────────────┤
│  Kandidaten                                          Autonomy: Risk-Off ●    │
│  9 aktiv · 3 candidate · 2 evidence_pending · 1 risk_warn   [Phase ▾][Sort ▾]│
├────────────────────────────────────────────────────────────────────────────┤
│  [Suche 🔍]            [Phase ▾] [Playbook ▾] [Scout ▾] [⚑ requiresHuman]    │
├──────────────┬────────┬──────────┬─────────┬──────────┬──────────────┬──────┤
│ Phase        │ Symbol │ Playbook │ Scout   │ Risk      │ Lebt weil    │  ⚑  │
├──────────────┼────────┼──────────┼─────────┼──────────┼──────────────┼──────┤
│ ● candidate  │ NVDA   │ Breakout │ support │ —         │ Trend+Vol ok │      │
│ ● evid_pend  │ AAPL   │ Breakout │ abstain │ pending   │ wartet Quelle│  ⚑  │
│ ● risk_warn  │ TSLA   │ Pullback │ support │ warn: ATR │ Setup ok     │  ⚑  │
└──────────────┴────────┴──────────┴─────────┴──────────┴──────────────┴──────┘
```

### Page Header

- **Titel:** „Kandidaten".
- **Autonomy-Pill** (rechts oben): `Risk-On` / `Neutral` / `Risk-Off` — gespiegelt aus dem
  Autonomy-Gate. Reiner Status (siehe §7).
- **Zähler:** aktiv gesamt · Aufschlüsselung nach Phase.
- **Sekundäraktionen:** Phasen-Filter, Sortierung. **Kein** „+ Hinzufügen" — Kandidaten entstehen nur
  durch die Setup-/Scout-Erkennung.

---

## 4. Tabelle / Spalten

| Spalte | Quelle | Sortierbar | Beschreibung |
|--------|--------|------------|--------------|
| Phase | `currentPhase` | ✅ | Phasen-Pill mit Farbe (s. u.). **Standard-Sortierung** nach `priority` (hardest gate oben). |
| Symbol | `identity.symbol` | ✅ | Ticker (mono), klickbar → Ticket-Akte. `Synthetic`-Badge falls zutreffend. |
| Playbook | `thesis.playbook` | ✅ | Setup-/Playbook-Typ (Breakout, Pullback …) + Richtung (▲/▼). |
| Scout | `council.verdict` (mode=scout) | ✅ | support / warn / abstain / block — Agentenrat-Evidenz. |
| Risk | `riskPreflight.state` | ✅ | `—` / `pending` / `warn` / `blocked` (read-only, mit Reason-Code-Kurz). |
| Lebt weil | `continuationReason` | – | Kurzbegründung, warum das Ticket weiterläuft. |
| Stirbt wenn | `deathCondition` | – | (optional einblendbar) Abbruchbedingung. |
| Erkannt | `refs`/Snapshot-Zeit | ✅ | Relativzeit („vor 4 min") — Frische. |
| ⚑ | `currentPhase.requiresHuman` | ✅ | Governance-Marker → Kandidat für Exception Inbox. |

### Phasen-Farbskala (Pill)

| Phase | Farbe | Bedeutung |
|-------|-------|-----------|
| `candidate` | `info` | sauberer Kandidat, läuft |
| `evidence_pending` | `neutral` | wartet auf Evidenz |
| `risk_warn` / `evidence_blocked` | `warning` | braucht Aufmerksamkeit |
| `risk_blocked` / `scout_rejected` / `gatekeeper_blocked` | `danger` | gestoppt vor Ausführung |

> **Kein** frontend-berechneter Score / CRV / Positionsgröße als Autorität. Falls der Snapshot einen
> Score liefert, wird er nur als **gespiegelter** Backend-Wert angezeigt, nicht neu gerechnet.

### Zeilen-Interaktion

- **Klick auf Zeile** → **Ticket-Akte** (Drawer rechts).
- **`requiresHuman`-Zeilen** erhalten den ⚑-Marker und einen dezenten Akzentbalken.
- **`risk_blocked` / `scout_rejected`** → ausgegraut, aber inspizierbar (nie versteckt, wenn im Filter).
- **Live-Update** → Phasen-Pill wechselt live bei Phasenänderung.

---

## 5. Ticket-Akte (Detail-Drawer)

Klick auf eine Zeile öffnet die **Ticket-Akte** — eine *Datei, keine Zeile* (Contract: „a file, not just a
row"). Sie macht den **Gate-Stack** transparent: *warum lebt das Ticket, was würde es töten, wer muss prüfen.*

```
┌──────────────────────────────────────────────┐
│ NVDA · Breakout ▲        Phase: candidate [✕] │
│ thesisId: th_8842   ·   origin: radar         │
├──────────────────────────────────────────────┤
│  LEBT WEIL   Trend intakt, Volumen-Confirm ok │
│  STIRBT WENN Schluss unter Range-Low / Risk-Block │
├──────────────────────────────────────────────┤
│  GATE-STACK                                   │
│  Radar       ✓ aktiv                          │
│  Gatekeeper  ✓ frei                           │
│  Scout       ◑ support (2/3, 1 abstain)       │
│  Evidence    ◑ 1 Quelle pending               │
│  Risk        — noch nicht ausgeführt          │
│  Autonomy    ⓘ Risk-Off (Hinweis)             │
│  Execution   – kein Intent                    │
├──────────────────────────────────────────────┤
│  SCOUT-COUNCIL (Agentenrat · mode=scout)      │
│  Verdikt: support   Reason: trend_confirm,    │
│  vol_confirm   ·   Abstain: macro_agent       │
│  [Trace im Agentenrat ansehen →]              │
├──────────────────────────────────────────────┤
│  RISK-PREFLIGHT (read-only)                   │
│  Status: noch nicht ausgeführt                │
│  (keine Frontend-Berechnung — Backend ist     │
│   maßgeblich für Risiko)                      │
├──────────────────────────────────────────────┤
│  FEHLENDE EVIDENZ                             │
│  • risk_preflight   • decision_chain (teilw.) │
├──────────────────────────────────────────────┤
│  [Zur abgesicherten Fläche →] [⚑ an Inbox]    │
└──────────────────────────────────────────────┘
```

### Drawer-Inhalt

1. **Kopf:** Symbol, Playbook + Richtung, aktuelle Phase, `thesisId`/`origin`, `Synthetic`-Badge.
2. **Lebt weil / Stirbt wenn:** `continuationReason` und `deathCondition` prominent.
3. **Gate-Stack:** Radar → Gatekeeper → Scout → Evidence → Risk → Autonomy → Execution (→ Manager/Review,
   sofern relevant), jeweils mit Status-Symbol und Kurzbegründung. Fehlende Gates erscheinen als
   `—`/`nicht verfügbar`, **nie** als erfundenes Ergebnis.
4. **Scout-Council:** Verdikt, Rollen-Zusammenfassungen, Reason-Codes, Abstentions — als **Evidenz**.
   Kein roher Modell-Gedankengang. Link in den Agentenrat-Trace.
5. **Risk-Preflight (read-only):** Status (`—`/`pending`/`warn`/`blocked`) + Reason-Codes + Refs, exakt
   wie im Snapshot. Hinweis, dass nicht neu gerechnet wird.
6. **Fehlende Evidenz:** explizite Liste aus `missingEvidence[]`.
7. **Aktionen:** siehe §6.

---

## 6. Aktionen — read-only + Navigation

Der Screen enthält **keine** mutierenden Trade-Controls. Erlaubt sind:

| Aktion | Beschreibung | Contract-Bezug |
|--------|--------------|----------------|
| Ticket-Akte öffnen | Drawer mit Gate-Stack | inspect |
| **Zur abgesicherten Fläche →** | Navigation zur bestehenden, guarded Order-/Execution-Fläche (dort findet die eigentliche Freigabe statt) | „Navigation to existing guarded surfaces is allowed" |
| In Agentenrat-Trace springen | `refs` → Council/Decision-Chain | audit |
| **⚑ An Exception Inbox** | Ticket als Governance-Fall melden (z. B. `risk_warn` ohne klare Policy) | Exception Inbox |
| Für Review markieren | bei geschlossenen/blockierten Pfaden | Review |

> **Verboten auf diesem Screen:** buy, sell, order, deploy, start paper, suspend, stop, „take trade
> anyway" — und jede Frontend-Berechnung von Risiko/Positionsgröße.

### Auflösung „Freigabe bleibt möglich"

Die ursprüngliche Idee einer Inline-Order-Freigabe entfällt — der Contract verbietet Mutation auf dem
Ticket-Board. **Stattdessen:** Der Screen führt per *„Zur abgesicherten Fläche →"* zur bestehenden,
guarded Execution-Oberfläche, wo die Freigabe regulär (mit Risk-Preflight als Autorität) passiert. Die
Navigation bleibt **immer** möglich; gesperrt wird sie nicht.

### Harte UI-Regeln (aus den Hard Gates)

- `risk_blocked` → **keine** Execution-Aktion sichtbar/auslösbar (auch keine Navigation, die als
  Resend-Autorität verstanden werden könnte).
- Agentenrat-Verdikt überschreibt **nie** Risk-Preflight, Suspend, Execution-Reject oder Operator-Locks.
- `synthetic` → read-only, nie als Beweis „Trade erlaubt".
- Unmanaged/manuelle Position, die hier auftaucht → als `exception` markieren und in die Inbox leiten.

---

## 7. Autonomy / Risk-Off — nur Hinweis

Risk-Off ist hier ein **Autonomy-Gate-Status**, kein UI-Lock.

| Autonomy | Verhalten im Screen |
|----------|---------------------|
| **Risk-On** | Autonomy-Pill grün, keine Banner. |
| **Neutral** | Pill neutral, dezenter Hinweis. |
| **Risk-Off** | **Info-Banner** oben: *„Risk-Off — Gatekeeper rät von neuen Einstiegen ab."* Pill rot. **Keine** Button-Sperre — Inspektion und Navigation bleiben voll möglich. |

> Bewusste Entscheidung: Der Screen *informiert* über Risk-Off, blockt aber nichts. Die tatsächliche
> Einstiegssperre liegt im Gatekeeper/Risk-Preflight des Backends — nicht in der read-only UI.
>
> `suspended` (System/Symbol/Autonomy) wird ebenfalls als Status gezeigt; betroffene Tickets bleiben
> inspizierbar, transiente Runtime-Health überschreibt aber keine geschlossenen Review-/Archiv-Zustände.

---

## 8. Agentenrat-Boundary in der UI

**Zeigt** (als Evidenz/Audit): `decision_chain`-Stufen, Reason-Codes, zusammengefasste Votes/Abstentions/
Blocker, `action_intent`, `playbook`, `thesis_stage`, Source-Health, Council-Mode = `scout`.

**Zeigt nie:** eine neue Frontend-Risikoberechnung, Bypass von Risk-Preflight, Override von
Execution-Reject, rohen privaten Modell-Gedankengang, direkte Trade-Mutation.

---

## 9. Exception Inbox-Anbindung

`requiresHuman`-Tickets sind die Brücke zur **Exception Inbox**. Typische Governance-Fälle aus dem
Kandidaten-Vorfeld:

- Risk-Warnung ohne klare Policy.
- Config-Mismatch / unklare effektive Config.
- Evidence-Quelle down / Source-Health degraded.
- Agentenrat-Uneinigkeit, Abstention oder unvollständige Confirm/Veto-Evidenz.
- unerwartete/manuelle/unmanaged Position, die als Kandidat erscheint.

Die Inbox navigiert zu Operations / Agentenrat / Risk / Config / Testcenter — **ohne** „take trade
anyway"-Controls.

---

## 10. Leerzustände

| Zustand | Darstellung |
|---------|-------------|
| Keine Kandidaten (`no_ticket`) | „Aktuell keine aktiven Setups" + „Das System prüft Radar/Watchlist laufend." |
| Keine Treffer (Filter) | „Keine Tickets für diese Phasen/Filter" + `Filter zurücksetzen`. |
| Nur `missingEvidence` | Ticket sichtbar mit expliziter Lücken-Liste statt erfundenem Status. |
| Risk-Off, keine Kandidaten | Info-Banner + „Bei Risk-Off entstehen seltener neue Setups." |
| Ladefehler | Fehler-Box + `Erneut versuchen`. |

---

## 11. Echtzeit-Verhalten

- **Neue Tickets / Phasenwechsel** erscheinen live (WebSocket/Snapshot-Refresh) mit Highlight-Puls.
- **Phasen-Eskalation** (z. B. `candidate` → `risk_warn` → `risk_blocked`) aktualisiert Pill + Sortierung
  live (hardest gate wins).
- **Scout-Verdikt-Updates** ändern die Scout-Spalte live.
- **Autonomy-Wechsel** schaltet Banner/Pill sofort um (ohne Buttons zu sperren).
- **Dedup:** Mehrere Evidenz-Quellen für dieselbe `thesis_id` bleiben ein Ticket in höchster Phase.

---

## 12. Offene Fragen

- Soll der Phasen-Filter `risk_blocked`/`scout_rejected`/`gatekeeper_blocked` standardmäßig ein- oder
  ausgeblendet sein?
- Wie viel Risk-Preflight-Detail (Reason-Codes, Refs) zeigen wir inline vs. erst in der Risk-Fläche?
- Eigener Tab in der Akte für die `decision_chain`-Historie (Verlauf der Stufen)?
- Wann genau wird ein Kandidat automatisch in die Exception Inbox eskaliert (Schwellen je Phase)?
- Benachrichtigung bei neuem `candidate` mit `requiresHuman` = true?

---

## 13. Querverweise

- **Contract (Grundlage):** `../docs/trade-ticket-lifecycle-contract.md` (Phasen, Read-Model, Hard Gates)
- Vorher: `03-watchlist.md` (`watch`-Phase) *(noch offen)*
- Nachher: Execution / Positionen — `execution_*`, `position_*` *(noch offen)*
- Exception Inbox / Agentenrat / Risk-Fläche *(noch offen)*
- Design-System: `01-design-system.md` *(noch offen)*
- Glossar: `glossary.md` *(noch offen)*
