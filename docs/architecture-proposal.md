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
| M4 | Generation: deterministic first, declared sampling later | every generated character carries its calculation record |
| M5 | V9 records connected to the admission interface when it exists; IRRS verification | IRRS recomputes; no self-certification |

Each milestone is reviewed by both teams before the next begins (README §6).

## 7. What we are asking of Robin's team

1. Review the five components and the proposed ownership.
2. Origin and Verifier rulings on Q1–Q5.
3. Your view on the corpus for M0. We suggest a public-domain novel with a fixed, hashed edition.
4. Whether IRRS should move to the V9 kernel before M1, or M1 should run on the V8 pin with that recorded as OPEN.

---

*Bounded over D. No claim beyond D.*
