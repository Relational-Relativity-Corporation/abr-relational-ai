# Architecture proposal: a tiny relational language model (attempt 5)

**Status:** SPEC PROPOSAL for joint review. Nothing here is decided until both teams agree. Its deliverable is a set of
jointly agreed specifications (§1.1); no model code is written until the spec it depends on is marked AGREED by both teams.
**From:** Bruce's team, answering README §5 ("Bruce must be invited to participate in defining the project architecture and the
interface between the proposed components before implementation begins") and §8 (open architectural questions).
**Kernel reference:** `abr-kernel` @ `84e7f8d` (Kernel V9, Milestone 3 specification). Operator forms are unchanged from V8.
This proposal changes no operator and closes no OPEN condition. Those belong to Origin and the Verifier.

---

## 1. What we propose to specify

The goal is a jointly agreed specification for a small language model over characters in which **every prediction traces** from a declared observed text,
through a declared mapping M and the frozen kernel operators, to a declared projection C. Nothing in the kernel path
is learned, tuned or smoothed. The one parameter that is steered (§7) belongs to C, outside D, and every setting of it is
recorded as a declaration. The intended end product is tiny: one Rust crate, depending only on the kernel.

**"Training" is accumulation.** The model is trained by admitting declared observations of a text, in the text's own
reading order, into a persistent record. The record *is* the trained model. Every relational state is re-derived from it
through the kernel. There are no gradients and no learned weights. A prediction can only use relations that were observed
before it, so the model is causal by construction rather than by a mask.

**The goal beyond prediction is self-tuning to near criticality** (§7): the model should sit at criticality because of how it is
built, not because something tuned it there. This is a difficult ask, and it is the part Bruce proposes to take on as his
main job in this collaboration.

**Attempt 5 is a learning instrument.** Its deliverables include recorded findings (what worked, what failed, and why) that
feed the specification of a next version. The aim of that next version is to be competitive with, and eventually to
outperform, conventional language models on declared comparators. Attempt 5 itself promises no performance; it promises
traceable results and an honest findings log.

### 1.1 Process: specs before code, tests before code

Robin and Bruce agree on specifications before any code is written. At this stage we are only building specs. Robin's team
has built four language models in this line and leads the core language-model specs (record, M, kernel use, C, evaluation);
Bruce's team proposes the added features (the criticality objective and the navigator, §7) for joint agreement. §3–§7 are
our proposed first drafts, not decisions. The pipeline from spec to code, as we propose it:

1. **Spec PROPOSED.** One document per component and per interface: the observation record (§3.1); M, one per candidate
   (§3.2); kernel use (§3.3); C (§3.4); evaluation (§3.5); the record↔kernel and kernel↔C interfaces; the navigator (§7.4); and
   one that collects the Origin rulings on Q1–Q5 as they land. A spec states what the component takes, produces and declares
   (keeps and discards), what is NOT EVALUATED, and the test that would show it wrong. Each lives on its own
   `planning/spec-<name>` branch and goes to `main` by PR (README §9).
2. **Spec AGREED.** Both teams review and record approval in the PR; the header is marked `AGREED <date>` with both names.
   One open objection holds it at PROPOSED. A later change is a new revision by the same route, marked `AGREED rev N`; never
   an edit in passing, never from inside an implementation PR.
3. **Tests written from the spec, before any code.** The test harness and the tests are derived from the agreed spec, not
   from code. Both teams review that the tests encode the spec and that each one can fail. They are committed and **seen RED**
   against an empty or stub implementation. The harness is itself a deliverable: kernel-conformance tests, causal-evaluation
   tests (no target leakage; causal by type), declaration and provenance checks, and the criticality measurement checks
   (estimator, shuffled-text null, bias floor). Editing a test afterwards needs an explicit, recorded reason.
4. **Plan locked.** Implementation starts only from a written plan that cites the agreed specs and names the tests it must
   turn green; it never starts from a conversation. The plan is read from the implementer's chair by a reviewer who marks
   every place the implementer would have to guess, dry-run where useful, revised until it can be executed mechanically, then
   frozen. A change is a new revision.
5. **Code until GREEN.** The implementer, on a `feature/*` branch opened only now, writes code until the named tests pass. It
   is deliberately not asked to be clever: it implements exactly the plan. If it must exercise much judgement, the plan was
   poorly written and the fix is to the plan, not to the implementer's discretion. It invents no tests and widens no scope.
   When it stops or gets stuck, the plan is diagnosed; in our experience nearly every stop has been a plan error.
6. **Verified and published.** The implementer commits and reports red/green per test by name. A second party verifies the
   red-then-green record and makes the publication decision (the push). A fresh agent with no context may serve as the
   acceptance check for usability and documentation.
7. **Findings logged.** Each milestone closes with a short record of what worked, what failed and why, kept in the repository
   and read before the next spec is drafted.

Where a test cannot be written in advance because the expected value is unknown (an exploratory measurement), the spec says so
per item and a **pre-registered prediction** stands in its place, as P1–P4 do (§7.4). The shared-build pipeline (§8) is
infrastructure, set up before step 3, and builds no model code. Spec order: record → M (M-a, M-b, M-c) → kernel use → C →
evaluation → interfaces → navigator; the Origin-rulings spec runs alongside and gates M and kernel use. This is the method
Bruce's team works by ("plan first, generate later"); we offer it for joint adoption, not as a condition.

## 2. Lessons from the four earlier attempts that shape this one

We read the earlier repositories in this line (relational attention, transformer integration, the language-structure
measurement repositories, and relational memory traversal). The lessons that shape this proposal:

1. **Score causally, in the repository, from the first commit.** In the transformer-integration model, the logit at
   position *i* receives the target token x_{i+1} through edge (i, i+1) (`op_b` adds the successor edge's field). Retrained
   and scored with the target unavailable, the model came out worse than a unigram baseline. We propose to submit the causal
   check as a separate `eval/*` contribution so both teams can reproduce it.
2. **No learned maps in the computation path.** Where they existed (embeddings, linear layers, an output matrix), they
   carried the computation and the operators did not.
3. **Declare M before A.** Index-modulo encodings form a ring, and PCA or standard-deviation scaling are undeclared transforms.
4. **Accumulators with thresholds and clamps are control.** The kernel already provides a persistence form (phase 2,
   E_current − E_prior), and we use it instead.
5. **A memory with no declared projection emits nothing.** C must be designed, not deferred.
6. **One repository.** A FAIL is a commit, not a reason to start over.

## 3. Architecture

Five components. Each is labelled as the kernel, a declared mapping M, or a declared projection C.

| # | component | standing | owner (proposed) |
|---|---|---|---|
| 1 | Observation record | the persisted declared observations, i.e. the trained model | Bruce's team (memory lane) |
| 2 | M: record → declared relations + NodeField | declared mapping | joint; Origin authorizes |
| 3 | Kernel: phase 1 `operator_e`, phase 2 `operator_e_v5` | frozen operators | Robin's team (kernel, IRRS) |
| 4 | C: kernel field → next-character scores | declared projection (application layer) | joint |
| 5 | Evaluation against comparators | outside D, declared as such | Bruce's team |

### 3.1 Observation record (memory and persistence)

- An append-only record. One entry is one **process step** of the observed text, a declared segment such as one line of a
  declared edition. Each entry carries its declaration record: corpus identity and hash, character span, the reading-order
  basis, the M identity, and whether the observation is simulated or empirical.
- Nothing else persists. In particular, kernel outputs are **never stored and reloaded**: the prior for step t+1 is
  re-derived by running the kernel on the record through step t. So `PersistenceState::from_prior` only ever receives
  kernel output, by construction, and needs no provenance check on a reloaded field. This is our proposed answer to
  OPEN E-1 and to README §8's "how is prior relational evidence represented and transferred across computation cycles?"

### 3.2 M: declared relations and the node field

- **Loci:** character types (one node per observed character). The text currently being read is a **path of positions**,
  each bound to its type. Positions alone cannot carry values: a code point is a label, not a quantity, so a directed
  difference of code points has no meaning.
- **Relations:** an edge c→d exists **if and only if** d was observed immediately after c in the admitted record.
  Provenance class: Continuation, within one observed text. Direction basis: the declared reading order of the edition.
  An edge with no admitting occurrence cannot be constructed, so an unseen transition is absent, not an edge with a small value.
- **Node values (the central OPEN design question).** Three candidates, to be compared in milestone M3. In each, a node
  value is a **count over the admitted record**: a declared reduction (M names the record span it reduces and what it keeps
  and discards). It is a projection through M, not an observable; the observable is the occurrence.
  - **M-a:** the cumulative occurrence count of each character type in the admitted record (keeps frequency; discards order
    beyond the declared relations). Under M-a every edge value A(x)[c→d] = n(c) − n(d) is fixed by the two counts; the
    reading order survives only in which edges exist.
  - **M-b:** loci are context types of a declared order k. A node value is the count of that k-character context, and an edge
    is an observed (k+1)-character continuation. It keeps more lexical identity and makes the graph larger.
  - **M-c:** loci are the observed contexts of every length, edges their observed one-character extensions (§7.4). It is the
    candidate under which a critical state is representable at all.
- M applies no smoothing, normalisation or learned map. Whatever M keeps and discards is written in its declaration.

### 3.3 Kernel

- Phase 1 at each step: `operator_e` (A → B → R) on the declared relations and node field. A → B → R is admissible only
  where M establishes an interior locus (s→t→u on independently admitted edges); the character graph does, through any
  character observed both after and before another. Where it does not, the primary kernel Δ → Σ applies and B is not evaluated.
- Phase 2 between steps t and t+1: `operator_e_v5` with the re-derived prior. As frozen, `operator_a_persistence` requires
  E_current and E_prior to have the same edge count. So phase 2 runs over a **declared pair restricted to the edges present
  at both steps**; the restriction is a declared projection of the step-(t+1) field, and M states what it drops. An edge
  first observed at step t+1 is outside the pair, and its persistence difference is **NOT EVALUATED**. No predecessor is
  fabricated and no operator is changed.
- ρ_base and χ₀ are Origin declarations with no defaults. Until they are declared, R is NOT EVALUATED (see Q5). The model
  declares one component and no component pairs, so cc is empty and the cross-topology term does not execute.

### 3.4 C: prediction

- Given the context path's last locus, C maps the kernel field on that locus's out-edges to one score per successor character.
  Which field C reads (R∘B∘A, or B∘A while R is NOT EVALUATED) is fixed by the ruling on Q5 and written in C's declaration.
- A character never observed after that locus has no edge, so its score is **NOT EVALUATED**: not zero, not smoothed. The
  model says what it has not seen.
- Any normalisation into a distribution happens inside C and is declared as an imported construct, stating what it keeps
  and discards.
- **The calculation record of a prediction is also its attention.** The V9 calculation record (§3, §5 of the V9
  specification) lists every contributing relation, value and parameter. For this model, that list is the explanation of
  why each character was predicted. We see this as the model's most distinctive property.

### 3.5 Evaluation

- Causal by type: prediction takes the current state and the context and has no access to the target. A target is admitted
  into the record only after it has been scored.
- We report coverage (the fraction of targets with an evaluated score), causal perplexity over evaluated targets, and the
  unigram and Kneser-Ney comparators on the same targets. Comparators and perplexity are **declared projections outside D**,
  conventional by design, and not kernel operators.
- Kernel conformance and task performance are **separate evaluations**. Passing one says nothing about the other.

## 4. A concrete V9 instance for the text domain

V9 Milestone 3 leaves the admission types OPEN. The text domain suggests concrete field values, offered as input to Origin and
the Verifier and not as a decision:

| V9 working field | proposed content for the text domain |
|---|---|
| `observable_source` | corpus identity + hash, and the character offset of the occurrence that admitted the edge |
| `direction_basis` | the declared reading order of the declared edition |
| `m_mapping_ref` | M identity + hash of its declaration |
| `declared_domain` | the admitted record through step t |
| NodeField declaration record | M identity, the locus (character type or context), and the record span the count reduces |

Until the V9 admission interface exists, our crate would keep a local record layer shaped to the specification and clearly
labelled as a proposal. IRRS currently pins the V8 kernel at `27b31ce`, so moving that pin to V9 is a question for Robin's team.

## 5. Questions for Origin

One question per item, each with both branches.

- **Q1 Cycles of independently observed transitions.** The character graph contains cycles (t→h→e→space→t), each edge admitted
  by its own occurrence. Nothing is its own predecessor through a computation. *If admissible:* character types are the loci.
  *If not:* positions become the loci, and type identity is carried by M through a declared lookup.
- **Q2 Reciprocal and zero edge images.** `declared_edge_image_admissibility_check` (`operators.rs:882`) rejects a field in which
  one edge's value is the negation of another's. When both c→d and d→c are observed, their directed differences are exact
  negatives. A self-loop (`ll`) or two equal counts gives 0, which is its own negation. Each direction has independent
  provenance (separate occurrences). *If the check governs admission:* M must break the symmetry by observation (M-b contexts
  may), and self-loops are excluded. *If provenance governs:* the check is a diagnostic.
- **Q3 Process step.** Is a declared segment of an observed text (a line of a written edition) the observed system's own process
  step, so that phase 2 applies? *If yes:* persistence as proposed. *If no:* phase 1 only, and continuity comes from the record alone.
- **Q4 Relations that grow.** Edges and nodes join when first observed, so the step-(t+1) field has more edges than the prior.
  The frozen `operator_a_persistence` requires equal edge counts. *If a declared restriction of the pair to the common edges is
  admissible:* phase 2 runs over that pair, and edges outside it are NOT EVALUATED (§3.3). *If not:* phase 2 is NOT EVALUATED
  at any step at which the edge set grew, and continuity comes from the record alone.
- **Q5 ρ_base and χ₀.** Which observable in a text domain supplies them? *If one can be declared:* full A → B → R. *If none:* R is NOT
  EVALUATED, which needs a ruling on whether C may read A → B without R.

## 6. Proposed sequence

Spec stages (S) come first and end in AGREED marks (§1.1). Code milestones (M) begin only after the specs they depend on are
agreed, and **each milestone's first deliverable is its tests, seen RED** before its plan is locked (§1.1 step 3). Each
milestone closes with a findings entry (§1.1 step 7).

| stage | deliverable | evidence |
|---|---|---|
| S0 | Spec: observation record; D (corpus, edition, hash, segment as process step); the shared-build pipeline (§8) green from both teams' pushes, building no model code | both approvals recorded; AGREED |
| S1 | Spec: M-a, M-b, M-c written out, direction basis, what each keeps and discards; Q1–Q5 submitted to Origin | AGREED; Verifier declaration stage: PASS, or OPEN listed |
| S2 | Spec: kernel use (phases, the common-edge restriction, which field C reads); C; evaluation and comparators; the two interfaces | AGREED |
| S3 | Spec: navigator (§7.4: readout, floor, sizes, classifier, bracket, band, escape rule) and the declarations log | AGREED; P1–P4 frozen in the spec before any run |
| M1 (needs S0–S2) | Tests first: record round-trip, provenance checks, bit-exact re-derivation. Then relational continuity (README §7): step t persisted and re-derived as the prior for step t+1, in two separate processes | tests RED then GREEN by name; the prior shown to change the result |
| M2 (needs S2) | Tests first: causal-by-type (a target leaked into a score fails), coverage accounting. Then the evaluation harness and comparators | RED then GREEN; no position scores differently because of the target |
| M3 (needs S1–S2) | Tests first: C conformance (unseen successor ⇒ NOT EVALUATED, declared normalisation only). Then predictive C; M-a vs M-b vs M-c compared causally | RED then GREEN; the comparator table published whatever it shows |
| M3c (needs S3) | Tests first: estimator against a known-MI source, shuffled null band, bias floor, classifier on synthetic exponential / power-law / plateau curves. Then M-c built; the navigator run open-loop on M-b and M-c | RED then GREEN; P1–P3 scored as pre-registered; misses published |
| M3d (needs S3) | Tests first: every navigator setting lands in the declarations log with provenance; the loop can touch only T. Then the navigator wired in | RED then GREEN; P4 scored |
| M4 | Tests first: each generated character carries a complete calculation record. Then generation: deterministic first, declared sampling later | RED then GREEN |
| M5 | V9 records connected to the admission interface when it exists; IRRS verification | IRRS recomputes; no self-certification |

Each stage is reviewed by both teams before the next begins (README §6). An agreed spec that a milestone shows to be wrong is
revised as a new agreed revision (§1.1), not patched in the implementation.

## 7. Objective: self-tuning to near criticality

### 7.1 The objective

**Bruce suggests that this system should self-tune to near criticality.** That means it should hold itself, as it learns, at
or just on the ordered side of the boundary between an ordered regime and a disordered one. It should get there and stay
there because of how it is built and steered, not because a person picked a parameter value.

This is probably the hardest part of the project and **Bruce's main job in the collaboration**; Robin's team is welcome to
take parts of it. The shared components (§3) are specified jointly either way.

### 7.2 What criticality means here, and why it is worth having

**Primer, for readers new to this literature.** A system with a control parameter typically has two phases. In the
**ordered** phase, correlations between distant parts do not decay to zero: the system is locked into a pattern (for a text
generator: repetition, loops). In the **disordered** phase, correlations decay exponentially with distance, with a finite
correlation length ξ: beyond ξ the parts are independent (noise). At the **critical point** between them, correlations
decay as a power law, so there is no scale ξ beyond which the system forgets; the response to inputs is largest there. In the
branching-process and neuroscience literature the same three cases are called super-critical (activity grows), sub-critical
(activity dies out) and critical; those labels do not map one-to-one onto ordered/disordered, so §7.4 names the phases for
this system by its own declared readout and uses "ordered" and "disordered" only in that declared sense.

Several independent literatures found that systems which process information work best at or near that boundary:

- **Computation** peaks at the "edge of chaos" (Langton 1990, for cellular automata, a claim that was contested on its
  evidence by Mitchell, Crutchfield & Hraber 1993; Bertschinger & Natschläger 2004, for recurrent networks).
- **Dynamic range** is maximised at criticality (Kinouchi & Copelli 2006, in a model). Cortical cultures show avalanche
  statistics with critical exponents (Beggs & Plenz 2003), with functional benefits reviewed by Shew & Plenz (2013).
- **Living systems sit near, not exactly at, criticality.** In vivo cortex looks driven and slightly sub-critical (Priesemann
  et al. 2014). Muñoz (2018) reviews the general picture and its caveats. This is why the objective says *near* criticality:
  a margin on one declared side keeps most of the benefit without the cost of the other phase.
- **Natural text has long-range correlations consistent with criticality, which is less than being critical.** Mutual
  information between characters at distance d, MI(d), looks like a power law over the measured range in large corpora (Lin &
  Tegmark 2017, Fig. 1: compared by eye, no exponent quoted, tails dominated by poems and markup). Earlier measurements report
  the same long-range structure without calling it critical (Ebeling & Pöschel 1994; Montemurro & Pury 2002), and Altmann et
  al. (2012) explain it by topic and keyword burstiness, a mixture rather than a critical point. So a power law in MI(d) is
  consistent with criticality but does not establish it; this proposal treats that as a stated limitation of the readout.
  The theorem side is firmer: any stationary irreducible aperiodic Markov process, of any finite order, has MI(d) decaying
  exponentially, a reducible or periodic one decays to a constant, and no Markov process gives a power law (Lin & Tegmark
  2017, Theorem 1 and App. B). That a finite-memory model cannot be critical is an inference from that theorem, not a
  measured result. Separately, Sacco, Sakthivadivel & Levin (2026) map any autoregressive model with a finite context window
  onto a one-dimensional local Hamiltonian and show it has no ordered phase at any finite temperature (Theorem 3, Corollary 2;
  Proposition 2 for causally masked attention). That is a result about order, not criticality; together the two say a fixed
  finite context admits neither. A model that holds itself near criticality is our candidate answer.

**How this relates to Robin's earlier work.** The framework's applied repositories detect a system's approach to an
instability early so that it can be avoided: the boundary as a hazard. This objective treats the same boundary as the
operating point to seek and hold; knowing early and reliably how close the system is to it is half of what a compass needs.

### 7.3 Approach

In one sentence: **the model is specified so that a critical state is representable (structure), and a one-parameter
navigator derived from the Criticality Compass is wired into the training loop to measure where the model sits relative to
the critical boundary and steer a declared parameter of C to hold it near there (navigation).** The kernel operators do not
optimise and are not asked to. The steps:

1. **Structure.** Use a mapping M under which a critical state can exist at all (M-c below: unbounded context). A fixed
   context order, sampled stationarily, is a finite-order Markov process and cannot be critical (Lin & Tegmark 2017,
   Theorem 1), so no amount of steering would help there.
2. **Measure, open-loop.** Run the navigator on the model across record lengths and settings of the one control parameter.
   It brackets the critical value from both sides, or reports that there is none.
3. **Wire in, closed-loop.** The navigator steers the declared control parameter of C as the record grows. It keeps the
   model on the declared side of the boundary found in step 2, by a declared margin.
4. **Verify.** Every setting the navigator makes is traceable in the record. Both teams check kernel conformance and task
   performance separately.

A note on the name. In self-organized criticality proper (Bak, Tang & Wiesenfeld 1987) the system reaches the critical point
with no tuned parameter, through slow driving and fast relaxation. Here the feedback is an explicit, declared instrument that
sets a parameter. The honest name for that is **self-tuning** (homeostatic regulation toward criticality), and we use it.

Key terms for anyone searching the literature: self-organized criticality, edge of chaos, critical branching, finite-size
scaling, closed-loop control of criticality, homeostatic regulation toward criticality.

### 7.4 Details

**What the Criticality Compass is for.** The Criticality Compass (CC) is Bruce's instrument for locating and classifying
critical behaviour from declared readouts. Its **design intent is to be wired into a system so that the system tunes itself
to criticality**. To date it has classified; this project is where the steering half of that intent would first be exercised.
Readers, human or model, should take it as the design intent, not a side experiment.

**Where the loop sits.** The kernel's operators are non-goal-directed and non-optimizing: nothing in A, B or R seeks
criticality, and nothing here asks them to. The navigator loop sits **outside the kernel and outside D**. It reads declared
reports about the model's output and sets one declared parameter **of C** (a sampling temperature T). It never sets a locus,
a relation, k, ρ_base or χ₀: a rule over operator outputs cannot declare a relation, and ρ_base and χ₀ are Origin declarations
from an observable, not settings. Because T lives in C, no operator ever receives it, so the kernel's non-optimizing property
is preserved by construction, not by the navigator's restraint. Each setting enters the record as a **declaration entry**
with its provenance (the reading that chose it), in a declarations log kept apart from the observation entries; it never
enters as an observation and never edits a field the kernel computed.

**Working hypothesis: the text is the critical system, and a model that records its structure without truncating it
inherits that criticality.** By §7.2, a fixed context order (M-b), sampled stationarily, cannot be critical whatever k is
chosen. Truncation is the distortion.

**Proposed M-c: an unbounded context tree.** Loci are the contexts of every length that occur in the admitted record. An edge
runs from a context to each one-character extension of it that was observed. C reads the longest context in the record that
matches the end of the current path (the longest-match rule is C's declared choice, not M's). This is the variable-order
Markov family (Willems, Shtarkov & Tjalkens 1995; Ron, Singer & Tishby 1996; Begleiter, El-Yaniv & Yona 2004) with counts in
place of weights; none of those papers measures correlations, so nothing about criticality is inherited from them. There is
no k to choose: the depth at which contexts become unique moves outward with the record length on its own. For the Origin
questions: a tree has no cycles, so Q1 does not arise for M-c; each edge has one direction, so the reciprocal half of Q2 does
not arise; the zero half remains, because a context that always continues the same way gives a zero directed difference. One
trap must be declared away: at a unique context, longest-match prediction copies the record verbatim from that point on, and
copied text has the corpus's MI(d) trivially. C therefore needs a declared **escape rule** (for example, read the longest
matching context with at least two observed continuations) before P2 can mean anything (§7.5).

**The measurement, as declared.** Every quantity below is a declared report outside D, computed inside C or the navigator.

- *Readout.* MI(d) = H(X_t) + H(X_{t+d}) − H(X_t, X_{t+d}) over characters at distance d, in bits, with the three entropies
  estimated by the Grassberger digamma estimator (Grassberger 2003, as used in Lin & Tegmark 2017 App. D and Shen 2019).
  Alphabet size K and the number of pairs are stated with every curve (Paninski 2003).
- *Floor.* The same estimator on a declared surrogate: the text with its characters shuffled, which destroys all order and
  keeps the unigram counts. Its 95% band over a declared number of shuffles is the null. The plug-in bias is of order
  (K−1)²/(2 N_pairs ln 2) bits, positive for MI: about 6×10⁻³ bits at K = 80 and 7×10⁵ pairs, which is where the published
  text curves sit near d ≈ 100. No paper in the cluster reports a shuffled null for MI(d); we propose it as a declared
  comparator. A tail is a claim only above the band, and on the corpus itself the measurable range of d is limited by it.
- *Two sizes.* N is the record length admitted (the system size; the finite-size-scaling axis). L is the length of generated
  text the readout is estimated on; L is chosen so that the floor sits well below the signal at the distances of interest,
  and is declared. N is the quantity that is varied; L is held large.
- *Cutoff.* d*(N) is the smallest d at which MI(d) enters the null band. For an exponential decay with correlation length ξ,
  d* grows at most as ξ·ln L and does not move with N. For a power law d^{−γ}, d* grows as a power of N. For an ordered state,
  MI(d) does not enter the band at all (a plateau).
- *Phase, named by the readout.* **Disordered:** d*(N) flat in N (exponential, finite ξ). **Critical:** d*(N) grows as a
  power of N, and the log-log slope of MI(d) over [d_min, d*] is stable in N within a declared interval. **Ordered:** a plateau
  above the band. The classifier, its thresholds and the declared interval are written before the run.
- *Control parameter.* T, the temperature at which C's normalised scores are sampled, with T = 1 the untempered counts. In the
  open-loop sweep k (M-b) is also varied, but as separate declared Ms, one declaration each, never as a steered quantity.

**A one-parameter navigator, derived from the Compass.** The full CC is built for **multivariate** criticality (an isolated
point versus a critical line across several control parameters). This model most likely has **one** control parameter that
matters, so we propose a small, separate **univariate variant of the Compass navigator**. Given the classifier above, it evaluates the phase at a declared grid of T values and at two or more N, and returns
the narrowest interval [T_lo, T_hi] with the ordered class at T_lo, the disordered class at T_hi, and the critical class at some
grid point between, at the declared resolution; it returns **"none"** when no T is classified critical at the largest N, or
when the class does not change across the grid. That is what **bracket from both sides** means here. **Near** is a declared
band: T held at T_lo + μ·(T_hi − T_lo) with the margin μ ∈ (0, ½) declared, that is, inside the bracket on its ordered side.
The full CC comes back into play only if the system turns out to be multivariate: a second parameter of C is steered at the
same time (for example an escape threshold), or P3 finds a critical line.

How this differs from a thermostat: a thermostat holds a given scalar at a given setpoint by the sign of the error. The
navigator's setpoint is measured by finite-size scaling in N, not given; it re-measures as N grows; and when it reports "none"
it sets nothing. Wired in, it is still a controller on one declared parameter of C, and we say so; it is not a controller on
anything inside D or the kernel. Acquisition, the hard part on laboratory data, is easy here: the kernel path is
deterministic, sampling in C uses a declared seed, any N can be admitted and any L generated. Navigator readings are
**declared reports about simulated output** (V9: declared as simulated) and lie outside D. Two stages, in order: open-loop
**measurement** (P1–P3), then **wired in**, steering T so the model holds itself inside the declared band as N grows (P4).

**Predictions, written before any run.** Each is scored with the declared estimator, floor, L, grid and classifier above.

- **P1.** M-b at any fixed k, sampled at T = 1: the disordered class at every N tried; d*(N) does not grow with N. Falsified by
  d*(N) growing as a power of N at any k.
- **P2.** M-c with the declared escape rule, at T = 1: the critical class, with d*(N) growing as a power of N and a log-log slope
  that agrees with the same estimate on the corpus within the declared interval. Falsified by the disordered class at the largest
  N, or by a slope outside the interval. The corpus slope is not a literature value: no paper quotes an exponent for text,
  so it is measured on our corpus, with the same estimator and null, and declared before P2 is scored.
- **P3.** Sweeping T on M-c: the navigator returns a bracket, not "none". Strong form: the bracket contains T = 1, as Nakaishi
  et al. (2024) found for large language models with a different readout (part-of-speech correlations, T_c ≈ 1). If the
  critical class spans a whole interval of T (a critical line) rather than a point, that is the more interesting result, and
  the point where the full CC is needed.
- **P4.** Wired in, the navigator holds the readout in the critical class on the ordered side of the bracket, within the
  declared band, at every N in a declared sequence; and it returns "none" for any M where P1 holds, rather than steering to a
  false bracket.

A miss is published like any other result.

### 7.5 Open design questions (for both teams, not Origin)

- The escape rule in C for M-c, and whether its threshold is a second control parameter.
- Whether MI(d) alone can carry the critical classification (§7.2:
  a power law is consistent with criticality, not proof of it; a second declared readout may be needed).
- The side of the bracket the margin sits on. The neuroscience margin (Priesemann et al. 2014) is on the
  short-correlation side; for a text generator the ordered side is repetition (Nakaishi et al. 2024's repetitive phase). The
  proposal says ordered side; the measurement in M3c should settle it.
- The normalisation inside C that turns scores into the distribution T acts on, declared with what it keeps and discards.

### 7.6 References

- Altmann, Cristadoro & Degli Esposti 2012, PNAS 109, 11582. doi:10.1073/pnas.1117723109
- Bak, Tang & Wiesenfeld 1987, Phys. Rev. Lett. 59, 381. doi:10.1103/PhysRevLett.59.381
- Begleiter, El-Yaniv & Yona 2004, J. Artif. Intell. Res. 22, 385. doi:10.1613/jair.1491
- Beggs & Plenz 2003, J. Neurosci. 23, 11167. doi:10.1523/JNEUROSCI.23-35-11167.2003
- Bertschinger & Natschläger 2004, Neural Comput. 16, 1413. doi:10.1162/089976604323057443
- Ebeling & Pöschel 1994, Europhys. Lett. 26, 241. doi:10.1209/0295-5075/26/4/001
- Grassberger 2003, arXiv:physics/0307138
- Kinouchi & Copelli 2006, Nature Phys. 2, 348. doi:10.1038/nphys289
- Langton 1990, Physica D 42, 12. doi:10.1016/0167-2789(90)90064-V
- Lin & Tegmark 2017, Entropy 19, 299. doi:10.3390/e19070299; arXiv:1606.06737
- Mitchell, Crutchfield & Hraber 1993, Complex Systems 7, 89. arXiv:adap-org/9303003
- Montemurro & Pury 2002, Fractals 10, 451. doi:10.1142/S0218348X02001257
- Muñoz 2018, Rev. Mod. Phys. 90, 031001. doi:10.1103/RevModPhys.90.031001
- Nakaishi, Nishikawa & Hukushima 2024, arXiv:2406.05335
- Paninski 2003, Neural Comput. 15, 1191. doi:10.1162/089976603321780272
- Priesemann et al. 2014, Front. Syst. Neurosci. 8, 108. doi:10.3389/fnsys.2014.00108
- Ron, Singer & Tishby 1996, Machine Learning 25, 117. doi:10.1023/A:1026490906255
- Sacco, Sakthivadivel & Levin 2026, Phil. Trans. R. Soc. A 384, 20250011. doi:10.1098/rsta.2025.0011; arXiv:2501.13188
- Shen 2019, arXiv:1905.04271
- Shew & Plenz 2013, Neuroscientist 19, 88. doi:10.1177/1073858412445487
- Willems, Shtarkov & Tjalkens 1995, IEEE Trans. Inf. Theory 41, 653. doi:10.1109/18.382012

## 8. Shared build

**The machines, for readers outside Bruce's team.** Three names come up in our notes and conversations:

- **Porchlight:** the shared build and compute machine, a small Apple-silicon desktop. This is where the runner, the builds,
  the tests and every training run live, and where Robin's team would get access.
- **penguin:** Bruce's everyday workstation. It is slow, so it only plans, reviews and pushes. Nothing for this project runs
  on it.
- **jill:** an always-on home server that runs other services of Bruce's. It is a fallback for compute, and this project
  does not normally use it.

Bruce's team will provide Porchlight as a self-hosted GitHub Actions runner for this repository:

- Every **push** to a branch of this repository runs build, test, the kernel conformance tests and a causal-evaluation smoke
  test. Logs and evidence are posted as Actions artifacts that both teams can see.
- Only `push` triggers the runner. Pull requests from forks never execute on it. Contributions from either team come from
  named branches in this repository (README §9), so they build automatically.
- The runner has its own restricted account. Shell access for Robin's team can be arranged on request.

The pipeline comes first, in S0: both teams see a green build from their own push before any model code is written.

## 9. What we are asking of Robin's team

1. Agreement on the process in §1.1 (specs before code, tests before code, the AGREED gate, the findings log), then review of the five components and
   the proposed ownership, with Robin's team leading the core language-model specs.
2. Origin and Verifier rulings on Q1–Q5.
3. Your view on the corpus for M1. We suggest a public-domain novel with a fixed, hashed edition.
4. Whether IRRS should move to the V9 kernel before M1, or M1 should run on the V8 pin with that recorded as OPEN.
5. Agreement that self-tuning to criticality (§7) is a project goal, with a univariate Compass navigator wired in as the means. Bruce leads it,
   and your team is welcome to take parts. We also ask for an Origin view on M-c as a third candidate M, on recording
   navigator-chosen settings as declarations, and on the rule that the closed loop may set parameters of C only, never a
   locus, a relation, ρ_base or χ₀ (§7.4).
6. Confirmation that push-triggered builds on a shared runner suit your team's workflow.

---

*Bounded over D. No claim beyond D.*