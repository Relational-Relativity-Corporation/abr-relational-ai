# Architecture proposal: a tiny relational language model (attempt 5)

**Status:** PROPOSAL for joint review. Nothing here is decided until both teams agree.
**From:** Bruce's team, answering README §5 ("Bruce must be invited to participate in defining the project architecture and the
interface between the proposed components before implementation begins") and §8 (open architectural questions).
**Kernel reference:** `abr-kernel` @ `84e7f8d` (Kernel V9, Milestone 3 specification). Operator forms are unchanged from V8.
This proposal changes no operator and closes no OPEN condition. Those belong to Origin and the Verifier.

---

## 1. What we propose to build

The goal is a small language model over characters in which **every prediction traces** from a declared observed text,
through a declared mapping M and the frozen kernel operators, to a declared projection C. Nothing in the computation path
is learned, tuned or smoothed. The intended end product is tiny: one Rust crate, depending only on the kernel.

**"Training" is accumulation.** The model is trained by admitting declared observations of a text, in the text's own
reading order, into a persistent record. The record *is* the trained model. Every relational state is re-derived from it
through the kernel. There are no gradients and no learned weights. A prediction can only use relations that were observed
before it, so the model is causal by construction rather than by a mask.

**The goal beyond prediction is self-tuning to near criticality** (§7): the model should sit at criticality because of how it is
built, not because something tuned it there. This is a difficult ask, and it is the part Bruce proposes to take on as his
main job in this collaboration.

## 2. Lessons from the four earlier attempts that shape this one

We read the earlier repositories in this line (relational attention, transformer integration, the language-structure
measurement repositories, and relational memory traversal). The lessons that shape this proposal:

1. **Score causally, in the repository, from the first commit.** In the transformer-integration model, the logit at
   position *i* receives the target token x_{i+1} through edge (i, i+1) (`op_b` adds the successor edge's field). Retrained
   and scored with the target unavailable, the model came out worse than a unigram baseline. We will submit the causal
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
- **Node values (the central OPEN design question).** Two candidates, to be compared in milestone M3:
  - **M-a:** the cumulative occurrence count of each character type in the admitted record (keeps frequency; discards order
    beyond the declared relations).
  - **M-b:** loci are context types of a declared order k. A node value is the count of that k-character context, and an edge
    is an observed (k+1)-character continuation. It keeps more lexical identity and makes the graph larger.
- M applies no smoothing, normalisation or learned map. Whatever M keeps and discards is written in its declaration.

### 3.3 Kernel

- Phase 1 at each step: `operator_e` (A → B → R) on the declared relations and node field.
- Phase 2 between steps t and t+1: `operator_e_v5` with the re-derived prior. Edges first observed at step t+1 have no prior,
  so their persistence difference is **NOT EVALUATED**. No predecessor is fabricated.
- ρ_base, χ₀ and cc are Origin declarations with no defaults. Until they are declared, R is NOT EVALUATED (see Q5).

### 3.4 C: prediction

- Given the context path's last locus, C maps the kernel field on that locus's out-edges to one score per successor character.
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
- **Q4 Relations that grow.** Edges and nodes join when first observed. *If admissible:* the persistence difference is NOT EVALUATED
  on an edge with no prior. *If not:* phase 2 runs only over edges present at both steps.
- **Q5 ρ_base and χ₀.** Which observable in a text domain supplies them? *If one can be declared:* full A → B → R. *If none:* R is NOT
  EVALUATED, which needs a ruling on whether C may read A → B without R.

## 6. Proposed sequence

| milestone | deliverable | evidence |
|---|---|---|
| M0 | Declarations only: D (corpus, edition, hash, segment as process step), M-a and M-b written out, direction basis; Q1–Q5 answered | Verifier declaration stage: PASS, or OPEN listed |
| M1 | Relational continuity (README §7) on the text domain: step t persisted and re-derived as the prior for step t+1, in two separate processes | bit-exact re-derivation; the prior shown to change the result |
| M2 | Causal evaluation harness, comparators, coverage reporting | causal by type; no position scores differently because of the target |
| M3 | Predictive C; M-a vs M-b compared causally | the comparator table published whatever it shows |
| M3c | Criticality, measured (§7): M-c built; the univariate navigator built and run open-loop on M-b and M-c | P1–P3 scored as written, misses published |
| M3d | Criticality, wired in (§7): the navigator steers the declared control parameter | P4 scored; every setting traceable in the record |
| M4 | Generation: deterministic first, declared sampling later | every generated character carries its calculation record |
| M5 | V9 records connected to the admission interface when it exists; IRRS verification | IRRS recomputes; no self-certification |

Each milestone is reviewed by both teams before the next begins (README §6).

## 7. Objective: self-tuning to near criticality

### 7.1 The objective

**Bruce suggests that this system should self-tune to near criticality.** That means it should hold itself, as it learns, at
or just on the ordered side of the boundary between an ordered regime and a disordered one. It should get there and stay
there because of how it is built and steered, not because a person picked a parameter value.

This is a difficult ask, probably the hardest part of the project. We expect it to be **Bruce's main job in the
collaboration**, and if Robin's team wants to take parts of it, so much the better. The shared components (§3) are built
jointly either way.

### 7.2 What criticality means here, and why it is worth having

A system with a control parameter typically has two regimes. On the **ordered (sub-critical)** side, perturbations die out
quickly, correlations are short and behaviour is rigid. On the **disordered (super-critical)** side, perturbations run away
and behaviour is noise. At the **critical** boundary between them, perturbations neither die nor explode, correlations extend
over all scales (power laws rather than exponentials), and the system responds to inputs across the widest range.

Several independent literatures found that systems which process information work best at or near that boundary:

- **Computation** peaks at the "edge of chaos" (Langton 1990; Bertschinger & Natschläger 2004).
- **Dynamic range and sensitivity** are maximised at criticality (Kinouchi & Copelli 2006). Cortex shows critical avalanche
  statistics (Beggs & Plenz 2003), with functional benefits reviewed by Shew & Plenz (2013).
- **Living systems sit near, not exactly at, criticality.** In vivo cortex looks driven and slightly sub-critical (Priesemann
  et al. 2014). Muñoz (2018) reviews the general picture. This is why the objective says *near* criticality: a margin on the
  ordered side keeps the benefits without the risk of runaway.
- **Language itself is critical in this sense.** Mutual information between characters decays as a power law with distance
  in natural text. Any finite-order Markov model decays exponentially instead (Lin & Tegmark 2017). Separately, Sacco,
  Sakthivadivel & Levin (2026) argue that causally masked attention in decoder-only transformers has no ordered phase at
  all. A model that holds itself near criticality is our candidate answer to both.

**How this relates to Robin's earlier work.** The framework's applied repositories detect a system's approach to an
instability early, so that the instability can be avoided (cascade early warning, pre-critical structure before energy
release, domain-boundary approach). That work treats the critical boundary as a hazard to warn of. This objective treats
the same boundary as the operating point to seek and hold. The two fit together: holding a system near a boundary requires
knowing, early and reliably, how close it is. Early-warning detection is half of what a compass needs.

### 7.3 Approach

In one paragraph, for human and model readers alike: **the model is built so that a critical state is representable
(structure), and an instrument, a one-parameter navigator derived from the Criticality Compass, is wired into the training loop so that it measures where the
model sits relative to the critical boundary and steers a declared control parameter to hold the model near it
(navigation).** The kernel operators do not optimise and are not asked to. The steering happens outside the kernel, through
declarations that are recorded with their provenance. The steps:

1. **Structure.** Use a mapping M under which a critical state can exist at all (M-c below: unbounded context). A fixed
   context order cannot be critical, so no amount of steering would help there.
2. **Measure, open-loop.** Run the navigator on the model across record lengths and settings of the one control parameter.
   It brackets the critical value from both sides, or reports that there is none.
3. **Wire in, closed-loop.** The navigator steers the declared control parameter as the record grows. It keeps the model on
   the ordered side of the boundary found in step 2, by a declared margin.
4. **Verify.** Every setting the navigator makes is traceable in the record. Both teams check kernel conformance and task
   performance separately.

Key terms for anyone searching the literature: self-organized criticality, edge of chaos, critical branching, finite-size
scaling, closed-loop control of criticality, homeostatic regulation toward criticality.

### 7.4 Details

**What the Criticality Compass is for.** The Criticality Compass (CC) is Bruce's instrument for locating and classifying
critical behaviour. Its **purpose is to be wired into a system so that the system tunes itself to criticality**: the CC
reads where the system is and steers its declared control parameters onto the critical point or line. In other words it
navigates. It does not just observe. This project is the first place that purpose meets a system we built ourselves.
Readers of this document, human or model, should take it as the design intent, not a side experiment.

**Why it is hard, and where the loop sits.** The kernel's operators are non-goal-directed and non-optimizing: nothing in
A, B or R seeks criticality, and nothing in this design asks them to. The navigator loop sits **outside the kernel and outside D**.
It reads declared reports about the model's output and sets declared control parameters. Each setting it makes enters the
observation record as a declaration with its provenance (the navigator reading that chose it). It never enters as an observation,
and it never edits a field the kernel computed. The navigator also has to be a compass rather than a thermostat. A thermostat
holds one side of a threshold. A compass finds and follows a critical manifold that is approached from both sides, and it
has to say when there is none to find. Structure carries the other half of the job: M must be able to represent a critical
state at all, or there is nothing for the navigator to steer to.

**Working hypothesis: the text is the critical system, and a model that records its structure without truncating it
inherits that criticality.** Natural text shows mutual information between characters that decays as a power law in their
distance, while any finite-order Markov process decays exponentially (Lin & Tegmark 2017). So a fixed context order (M-b) is
sub-critical by construction, whatever k is chosen. Truncation is the distortion.

**Proposed M-c: an unbounded context tree.** Loci are the contexts of every length that occur in the admitted record. An edge
runs from a context to each one-character extension of it that was observed. Prediction reads the longest context in the
record that matches the end of the current path. There is no k to choose: the depth at which contexts become unique moves
outward with the record length on its own. Two side effects matter for the Origin questions. A tree has no cycles, so Q1 does
not arise for M-c. Each edge has one direction, so the reciprocal half of Q2 does not arise either. The zero half remains,
because a context that always continues the same way gives a zero directed difference.

**A one-parameter navigator, derived from the Compass.** The full CC is built for **multivariate** criticality: it tells an
isolated critical point from a critical line or walking behaviour across several control parameters at once. This model
most likely has **one** control parameter that matters at a time, so its criticality is probably univariate, and most of
the full CC would sit idle. We therefore propose a small, separate component: a **univariate variant of the Compass
navigator**. It brackets the critical value of one declared control parameter from both sides, using finite-size scaling in
the record length, and it reports "none" when there is no critical value to find. The component is generic and reusable
beyond this project. The full CC comes back into play only if the system turns out to be multivariate: a second control
parameter is steered at the same time (for example a sampling temperature together with context depth, or ρ_base and χ₀),
or P3 finds a critical line.

Acquisition is usually the navigator's hardest problem on laboratory data. Here the domain is **fully known**: the model is
deterministic, any record length can be admitted and any number of samples can be generated. So the navigator can acquire
everything it needs:

- size axis: the record length N;
- observable: the mutual information decay of generated text, against the same quantity on the corpus;
- control axis: one declared parameter at a time (k in M-b; a sampling temperature in C).

Navigator readings here are **declared reports about simulated output** (V9: declared as simulated) and lie outside D. Two stages,
in order: first the navigator is run open-loop to **measure** (predictions P1–P3 below); then it is **wired in**, so that it steers
the remaining control parameter and the model holds itself at criticality as the record grows (P4).

**Predictions, written before any run:**

- **P1.** M-b at any fixed k: mutual information decays exponentially (not critical).
- **P2.** M-c: power-law decay with the corpus's exponent, up to a cutoff distance that grows with N (finite-size scaling).
- **P3.** A sampling temperature in C: if it produces a transition (as reported for large language models by Nakaishi et al.
  2024), the navigator brackets it. If the critical side turns out to be a whole phase (a critical line) rather than a point, that is
  the more interesting result, and the point where the full CC is needed.
- **P4.** Wired in, the navigator keeps the model on the critical manifold found in P1–P3 as N grows, and it reports "no manifold"
  for any M where P1 holds, rather than steering to a false one.

A miss is published like any other result.

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

The pipeline comes first, as part of M0: both teams see a green build from their own push before any model code is written.

## 9. What we are asking of Robin's team

1. Review the five components and the proposed ownership.
2. Origin and Verifier rulings on Q1–Q5.
3. Your view on the corpus for M0. We suggest a public-domain novel with a fixed, hashed edition.
4. Whether IRRS should move to the V9 kernel before M1, or M1 should run on the V8 pin with that recorded as OPEN.
5. Agreement that self-tuning to criticality (§7) is a project goal, with a univariate Compass navigator wired in as the means. Bruce leads it,
   and your team is welcome to take parts. We also ask for an Origin view on M-c as a third candidate M, and on recording
   navigator-chosen settings as declarations.
6. Confirmation that push-triggered builds on a shared runner suit your team's workflow.

---

*Bounded over D. No claim beyond D.*