# ABR Relational AI

> **Relation is the observable primitive. The mathematics is what we construct from it.**

This project develops an AI architecture from observable relationships, using declared
mathematical mappings, traceable computational processes, and the Integrated Relational
Reference System (IRRS) as shared verification infrastructure.

---

## Contents

1. [Governing principle](#1-governing-principle)
2. [Objective](#2-objective)
3. [Declared mappings, mathematical construction, and provenance](#3-declared-mappings-mathematical-construction-and-provenance)
4. [IRRS — current capabilities and status](#4-irrs--current-capabilities-and-status)
5. [Proposed collaboration](#5-proposed-collaboration)
6. [Initial development sequence](#6-initial-development-sequence)
7. [First joint milestone — M1: Relational Continuity](#7-first-joint-milestone--m1-relational-continuity)
8. [Open architectural questions](#8-open-architectural-questions)
9. [Collaboration process](#9-collaboration-process)

---

## 1. Governing principle

**Relation is the observable primitive. The mathematics is what we construct from it.**

This principle governs the entire project.

Our requirement is that admitted computational results remain independently traceable to
observable relations. We do not prohibit conventional mathematics, prescribe an AI
architecture, or require particular algorithms merely because they are familiar or
unfamiliar. The governing constraint is provenance: a result is admitted when its
derivation is traceable through declared mappings and transformations to observable
relations.

---

## 2. Objective

We are building an AI architecture whose computational results are traceable to
observable relationships — not merely claimed to be, but demonstrably traceable through
a declared and independently reviewable chain: observable source → declared mapping (M)
→ relational computation → admitted result.

This is not a constraint on what mathematics may be used. It is a requirement about how
the derivation of results is documented and verified.

---

## 3. Declared mappings, mathematical construction, and provenance

Three elements together constitute an admissible result:

**Observable source.** A declared domain (D) with observable relations identified by
origin-authorized criteria. The physical basis of the observation is stated explicitly —
what was observed, by what method, with what known limitations.

**Declared mapping (M).** An authorized function from observable inputs to the
mathematical quantities submitted to computation. What M preserves and what it discards
are both declared. M is executed by the admission gate, not by the Generator.

**Traceable provenance.** For every admitted result, an independently reviewable record
establishes:
- Which observable sources were used
- What mapping was applied and by whom
- Whether Origin reviewed and authorized the declaration
- What mathematical operations were performed

A correct numerical result is not sufficient evidence of traceable provenance. The
derivation must be independently verifiable.

---

## 4. IRRS — current capabilities and status

**Repository:** https://github.com/Relational-Relativity-Corporation/abr-irrs

The Integrated Relational Reference System (IRRS) is our shared verification
infrastructure. It is operational and in active development. Its purpose in this project
is to establish and enforce traceability from computational results through declared
mappings and transformations to observable relations.

**IRRS is not an authority for prescribing which mathematical techniques developers may
use.** It is a traceability and admission system.

### Established capabilities

The following are implemented and evidenced by executable tests (commit `27b31ce`):

| Capability | Status |
|---|---|
| Kernel V8 operators: A (edge differences), B (successor accumulation), R (relational output) | Frozen reference |
| Kernel V8 operator: E (persistence, V5) | Frozen reference |
| Topology and field declaration (D, M) | Operational |
| ρ parameter declaration with provenance label | Operational |
| Observational comparison and correspondence checking | Operational |
| `EvaluationRecord` admission enforcement | Operational |
| Observable provenance — ADMITTED path (Origin-authorized record) | Operational |
| Observable provenance — OPEN path (basis not declared; awaiting authorization) | Operational |
| Observable provenance — REJECTED path (Failed EvaluationRecord blocks execution) | Operational |
| Trace report with provenance section | Operational |

### OPEN conditions

The following are not yet resolved. Results in these areas are available for
experimental use but not yet admitted under the full provenance requirement:

| Condition | Description |
|---|---|
| **OPEN E-1** | Session persistence API gap. `operator_e_v5` and `PersistenceState` exist in the kernel; the session layer does not expose persistence across calls. |
| **OPEN RhoMaxSelectionRule** | χ magnitude selection across components and incident edges unconfirmed (§11.2, §18 of the kernel declaration). |
| **OPEN RhoSaturatingForm** | Saturating form ρ = ρ\_base · χ / (χ₀ + χ) not derived from relational first principles (§11.2, §18). |

### What IRRS does not yet support

- Physical observational domain declarations (all current tests use synthetic fixtures)
- Session persistence across calls (OPEN E-1)
- AI-specific computation patterns (subject to M1 design)

Capabilities must be used with their verified status accurately represented. Do not
claim that a requirement is enforced unless code and execution evidence demonstrate
enforcement.

---

## 5. Proposed collaboration

This project brings together two human-led development teams. The assignment below is
a **proposal for discussion**, not a final decision. Both teams should have meaningful
architectural and programming responsibilities, including the ability to propose
mathematical approaches, implementations, and changes.

**Bruce must be invited to participate in defining the project architecture and the
interface between the proposed components before implementation begins.**

### Robin's team — proposed focus

Relational computation, observable-domain declarations, kernel integration, and IRRS
execution.

Responsibilities may include:
- Declaring observable domains (D) and mappings (M)
- Integrating IRRS admission and provenance verification
- Developing and running the kernel computation path (A → B → R)
- Producing and reviewing `EvaluationRecord` artifacts

### Bruce's team — proposed focus

Relational memory, persistence, context, and structural organization.

Responsibilities may include:
- Designing the memory and persistence layer (relevant to OPEN E-1)
- Developing the context and retrieval architecture
- Proposing the structural organization of relational state across computation cycles
- Defining the interface between the memory layer and the kernel computation path

### What is not yet decided

- Whether the proposed focus areas are the right division
- The interface between the computation and memory/persistence components
- Whether additional teams or collaborators will be involved
- The mathematical approaches each team will use within their areas

These questions are for Bruce and Robin to resolve together.

---

## 6. Initial development sequence

The smallest observable relational example is the correct starting point. Complexity is
introduced only as the observable evidence supports it.

**Sequence (proposed — subject to joint revision):**

1. **Declare a minimal observable domain.** Identify the smallest real or
   representative observable relationship that both teams can work from. Establish D
   and M. Verify that the domain satisfies the IRRS regional prerequisite (interior
   locus established).

2. **Run a single relational computation cycle.** Declare observable inputs, obtain an
   `EvaluationRecord`, have Origin review and authorize it, submit to the IRRS session,
   execute A → B → R, confirm the admitted result carries verified provenance.

3. **Demonstrate relational continuity.** Show that the result of one computation cycle
   can serve as traceable prior relational evidence for the next. This is M1. Requires
   resolving OPEN E-1 (session persistence API) or working around it with a declared
   interim design.

4. **Extend to memory and retrieval.** Once continuity is demonstrated, develop the
   memory and context layer (Bruce's team's proposed focus) with a declared interface
   to the kernel computation path.

5. **Evaluate and revise.** After each stage, both teams evaluate the results against
   the governing principle. Revise the architecture as the observable evidence warrants.

Each stage produces executable evidence reviewed by both teams before the next begins.

---

## 7. First joint milestone — M1: Relational Continuity

**Goal:** Demonstrate relational continuity across successive observations.

M1 is complete when the following are demonstrated in executable, reviewable evidence:

| Requirement | Description |
|---|---|
| Declared observable relationship | A domain D and mapping M identified and Origin-authorized. Inputs trace to declared observable sources. |
| Calculation derived from observations | A → B → R executed on declared inputs. Result is non-synthetic. |
| Traceable mathematical provenance | An Origin-authorized `EvaluationRecord` accompanies the result. The derivation chain is independently reviewable. |
| Preservation and retrieval of relational state | The result of one cycle is preserved and available as prior relational evidence for the next. |
| IRRS evaluation | The computation is evaluated through the IRRS capabilities applicable at the time of M1. OPEN conditions are accurately reported. |

**M1 does not require:** resolution of all OPEN conditions; a complete AI architecture;
or agreement on all mathematical approaches. It requires a minimal, honest, traceable
demonstration.

**The detailed design, implementation interface, and division of labor for M1 are open
for collaborative planning by both teams.**

---

## 8. Open architectural questions

These questions are for Bruce and Robin to resolve together. They are not decided by
this document.

**On the computation and memory interface:**
- What is the declared interface between the relational computation path (A → B → R)
  and the memory/persistence layer? What does each component expose to the other?
- How is prior relational evidence represented and transferred across computation
  cycles? What provenance does that transfer require?
- Who owns the declaration of what counts as a "prior" for OPEN E-1 purposes?

**On observational domains:**
- What is the smallest real observable relationship both teams can agree to work from
  for M1? Does it need to be physical, or is a rigorously declared representative
  example appropriate for the first milestone?
- How are observable sources identified, bounded, and distinguished from synthetic
  fixtures?

**On the AI architecture:**
- What mathematical approaches does each team propose for their area of responsibility?
- Are there approaches that raise provenance concerns under the governing principle?
  If so, how are those concerns stated and resolved?
- How is the boundary between the admitted computation path and experimental/OPEN
  computation paths marked in the architecture?

**On IRRS evolution:**
- Which OPEN conditions need to be resolved for M1, and which can remain OPEN with
  accurate documentation?
- When the IRRS capabilities evolve during development, what review process governs
  those changes?

---

## 9. Collaboration process

**GitHub workflow**

| Activity | Mechanism |
|---|---|
| Propose a change | Open a GitHub Issue describing the proposal, its motivation, and any provenance implications |
| Develop a change | Work on a named branch (e.g. `feature/m1-observable-domain`, `planning/memory-interface`) |
| Submit for review | Open a Pull Request against `main`; tag both teams; do not merge without review from both |
| Record evidence | Attach execution evidence (test output, trace reports, `EvaluationRecord` artifacts) to the relevant PR or Issue |
| Resolve OPEN conditions | Open a dedicated Issue; close it only when both teams agree the condition is resolved and execution evidence supports that claim |

**Branching conventions (proposed)**

- `planning/*` — architecture, design, and planning documents
- `feature/*` — implementation work
- `eval/*` — evaluation records and authorization artifacts
- `main` — reviewed, jointly accepted work only

**Evidence records**

Every admitted result should be accompanied by:
- The `EvaluationRecord` produced by `irrs_eval::Evaluator`
- Origin authorization (when applicable)
- The execution trace from `Session::trace_report`

Evidence records live alongside the code that produced them. They are not summaries —
they are the primary artifacts for independent review.

**First action**

Before any implementation begins: Bruce's team reviews this proposal and participates
in revising the architecture, the M1 design, and the division of responsibilities.
Both teams must agree on the interface between their proposed areas before development
proceeds.

---

## Status

| Item | Status |
|---|---|
| Governing principle | Established |
| IRRS — kernel V8 operators | Established (frozen) |
| IRRS — observable provenance enforcement | Established |
| IRRS — OPEN E-1 (session persistence) | OPEN |
| IRRS — OPEN kernel mathematical conditions | OPEN |
| Project architecture | Proposed — pending joint review |
| M1 design | Proposed — pending joint design |
| Team division of responsibilities | Proposed — pending Bruce's participation |
| Physical observational domain | Not yet identified |

---

*Metatron Dynamics, Inc. | Bounded over D. No claim beyond D.*
