# Research Companion Sheet — CS2

# 0. Quick Paper Identity Card

| Field | Detail |
|---|---|
| Problem Domain | Neural network robustness verification and certification |
| Paper Type | Mathematical / Theoretical + Algorithmic / Method + Experimental Empirical |
| Core Contribution | DeepPoly: a new abstract domain and transformers for proving robustness of deep networks |
| Key Idea (1-2 lines) | Represent each neuron with relational linear bounds plus concrete interval bounds, then propagate them with tailored transformers (affine, ReLU, sigmoid, tanh, maxpool) to over-approximate all perturbed inputs soundly. |
| Required Background | Neural network layers, adversarial examples, abstract interpretation, basic linear bounds |
| Primary Baseline | AI2 (Zonotope), Fast-Lin, DeepZ |
| Main Innovation Type | Domain design + transformer design + verification workflow + refinement for rotations |
| Difficulty Level | High |
| Reproducibility Level | Medium-High (open implementation and datasets reported, but implementation-level optimizations are important) |

# 1. Research Context and Core Problem

## 1.1 Exact Problem Formulation

- Input: a trained neural network and an adversarial region around an input image.
- Goal: prove that all points in that region map to the same class label.
- Region type in main setting: interval-bounded perturbations (commonly L-infinity style box constraints).
- Extended setting: rotation-based perturbations with linear interpolation, combined with pixel perturbation.

## 1.2 Why the Problem Exists

- Neural networks can change output class under tiny input changes.
- Safety-critical use cases need formal guarantees, not only empirical attack resistance.
- Brute-force checking all perturbed inputs is infeasible because the input space is combinatorially large.

## 1.3 Historical Gap

- Complete verifiers were precise but did not scale to larger networks.
- Scalable methods often lost too much precision.
- Some influential methods were not sound under floating-point arithmetic, causing reliability issues in guarantees.

## 1.4 Limitations of Previous Approaches

- SMT/MILP complete methods: strong guarantees but poor scalability on large architectures.
- Generic abstract domains: either too expensive (e.g., richer polyhedra) or too coarse (e.g., plain zonotope behavior in key cases).
- Fast-Lin style methods: efficient but limited architecture support and floating-point soundness issues.
- Earlier scalable sound methods: still noticeable precision drop when perturbation size grows.

## 1.5 Contribution Category

- Theoretical contribution: new abstract domain with invariants and soundness proofs.
- Algorithmic contribution: specialized abstract transformers and backsubstitution strategy.
- Verification strategy contribution: refinement/tracing for complex transformations.
- Empirical contribution: broad comparisons showing precision and scalability improvements.

### Why This Paper Matters

- It shows that precision and scalability can be improved together by domain-engineered abstractions.
- It makes certification practical on larger feedforward and convolutional networks.
- It extends robustness verification beyond standard pixel boxes to rotation with interpolation.

### Remaining Open Problems

- Runtime on some large convolutional architectures is still high.
- Guarantees are conservative because abstractions over-approximate.
- Complex real-world transformations beyond rotation remain challenging.
- Certification quality still depends on network architecture/training quality.

# 2. Minimum Background Concepts

## 2.1 Adversarial Region

- Plain definition: a set of allowed input perturbations around one image.
- Role in paper: this is the input set to certify.
- Why needed: robustness claim is always relative to a perturbation set.

## 2.2 Abstract Interpretation

- Plain definition: reason about many concrete states at once using a safe approximation.
- Role in paper: replaces impossible enumeration of all perturbed inputs.
- Why needed: gives sound proofs at scale.

## 2.3 Relational Bounds vs Interval Bounds

- Plain definition: relational bounds connect variables to previous variables; interval bounds are numeric min/max ranges.
- Role in paper: DeepPoly stores both for each neuron.
- Why needed: relational bounds improve precision, interval bounds improve speed and transformer design.

## 2.4 Transformer

- Plain definition: rule that updates abstract state when passing one network operation.
- Role in paper: custom transformers for affine, ReLU, sigmoid, tanh, maxpool.
- Why needed: generic transformers were either too weak or too slow.

## 2.5 Soundness Under Floating Point

- Plain definition: guarantee remains correct when computations use finite-precision arithmetic.
- Role in paper: authors explicitly address rounding effects.
- Why needed: practical neural inference uses floating-point, not real-number arithmetic.

# 3. Mathematical and Theoretical Understanding Layer

## 3.1 Core Abstraction Structure

- Intuition: each neuron gets two symbolic linear bounds (lower and upper) plus concrete scalar lower/upper bounds.
- What problem it solves: keeps enough dependency information without full polyhedral explosion.
- Practical interpretation: better precision than pure interval-style propagation, with manageable runtime.
- Limitation: still an over-approximation, so some true-robust cases remain unproved.

### Variable Meaning Table

| Symbol | Meaning |
|---|---|
| x_i | activation variable for neuron i |
| a_i^<= | symbolic lower relational bound for x_i |
| a_i^>= | symbolic upper relational bound for x_i |
| l_i | concrete lower bound for x_i |
| u_i | concrete upper bound for x_i |
| gamma(a) | concretization of abstract element a |

## 3.2 Domain Invariant

- Intuition: symbolic region must remain inside tracked interval box.
- Purpose: keeps transformer computations efficient and consistent.
- Assumption: all variables processed in order, bounded in practice for analyzed networks.
- Practical interpretation: interval bounds act as trusted envelopes around symbolic expressions.

## 3.3 ReLU Approximation Logic

- Intuition before equation:
- ReLU is exact when input is definitely non-positive or definitely non-negative.
- Hard case is when bounds cross zero.
- Problem solved: capture crossing-zero case with one lower relation and one upper relation to avoid combinatorial blowup.
- Design choice: choose between two candidate lower approximations by area criterion.
- Limitation: precision is sacrificed relative to richer convex hull encodings.

## 3.4 Affine Backsubstitution

- Intuition: do not estimate output bounds directly from local intervals; recursively substitute symbolic bounds backward.
- Problem solved: avoids loose bounds that destroy proof power.
- Practical interpretation: this is a key reason DeepPoly proves more properties.
- Runtime implication: dominant cost term in hidden layers; complexity grows with layer width and depth.

## 3.5 Soundness Theorems (What to Remember)

- ReLU transformer soundness proved.
- Sigmoid/tanh transformer soundness proved.
- Maxpool transformer soundness proved.
- Affine transformer sound and exact (in its abstract semantics) proved.
- Invariant preservation proved for all transformers.
- Floating-point soundness addressed by rounding-aware bound adjustments.

### Mathematical Insight Box

Keep one carefully chosen relational lower bound and one relational upper bound per neuron, then recover precision via selective backsubstitution. This combination gives a practical middle point between expensive precise polyhedra and fast but coarse abstractions.

# 4. Proposed Method and Framework (Most Important)

## 4.1 Overall Pipeline

1. Encode perturbation set as interval constraints on input neurons.
2. Convert network into assignment sequence (affine and activation/pooling assignments).
3. Propagate abstract state layer-by-layer using DeepPoly transformers.
4. Add output-difference constraints (target class vs other classes).
5. Check whether all required lower bounds are positive.
6. If not proved, optionally refine the input abstraction and retry.

## 4.2 Components and Data Flow

- Abstract state store:
- symbolic lower/upper linear forms per variable
- concrete lower/upper scalars per variable
- Transformer engine:
- affine transformer with recursive bound substitution
- ReLU/sigmoid/tanh/maxpool tailored transformers
- Property checker:
- output margin constraints through temporary affine variables

## 4.3 Simplified Pseudocode-Style Explanation

```text
initialize abstract_state from input intervals

for each network assignment in topological order:
	if affine:
		set symbolic bounds exactly to affine expression
		compute concrete bounds via recursive substitution
	if ReLU/sigmoid/tanh/maxpool:
		apply operation-specific safe linearization
		update concrete bounds

for each non-target output class c:
	create margin variable m_c = output_target - output_c
	propagate affine transformer

if all lower_bounds(m_c) > 0:
	return PROVED ROBUST
else:
	return NOT PROVED
```

## 4.4 Stepwise Critical Analysis

### Step A: Input Abstraction

- Why authors did this: direct encoding of perturbation region.
- Weakness: interval boxes ignore correlations between pixels.
- Improvement seed: use richer input set encodings (hybrid zonotope/polytope or learned perturbation structure).

### Step B: Layerwise Transformer Propagation

- Why authors did this: scalable compositional verification.
- Weakness: approximation error accumulates with depth.
- Improvement seed: adaptive precision by layer sensitivity.

### Step C: ReLU Crossing-Zero Handling

- Why authors did this: control exponential bound explosion.
- Weakness: lossy in ambiguous activation regions.
- Improvement seed: split only high-impact ambiguous neurons (guided partitioning).

### Step D: Output Margin Checking

- Why authors did this: classification robustness reduces to positivity of margins.
- Weakness: still one-vs-all; may miss tighter multi-class reasoning opportunities.
- Improvement seed: joint margin optimization constraints.

### Step E: Refinement for Complex Perturbations

- Why authors did this: single global approximation too coarse for rotation+interpolation.
- Weakness: runtime can increase significantly with refinement granularity.
- Improvement seed: adaptive batch sizing based on bound uncertainty.

# 5. Experimental Setup and Evaluation Design

## 5.1 Datasets and Samples

- MNIST (28x28 grayscale) and CIFAR10 (32x32 RGB).
- First 100 test images per dataset considered.
- Only correctly classified images were used for certification tasks.

## 5.2 Network Families

- Fully connected and convolutional models.
- Hidden units up to about 88K in reported set.
- Activation diversity: ReLU, sigmoid, tanh models.
- Defended and undefended variants (DiffAI and PGD adversarial training included).

## 5.3 Metrics and Protocol

- Main metric 1: percentage of properties proved robust.
- Main metric 2: average runtime.
- Perturbation families:
- L-infinity bounded perturbations across multiple epsilons.
- Rotation with interpolation plus perturbation.

## 5.4 Baseline Logic

- AI2: earlier abstraction-based verifier, less tailored transformers.
- Fast-Lin: layerwise linear approximation for ReLU feedforward; floating-point soundness limitation.
- DeepZ: specialized zonotope-based sound analyzer; key direct comparison point.

## 5.5 Compute and Hardware Assumptions

- Feedforward experiments: multi-core high-frequency CPU.
- Convolutional experiments: larger-memory multi-core server CPU.
- Both sequential and parallel forms discussed, with fairness notes in single-thread comparisons.

### Experimental Reliability Analysis

- Trustworthy:
- broad architecture variety
- defended vs undefended comparisons
- precision and runtime both reported
- cross-tool comparison on common settings
- Questionable or limited:
- sample count per dataset is modest (100 test images)
- no confidence intervals/statistical hypothesis testing reported
- runtime sensitive to implementation and hardware specifics

# 6. Results and Findings Interpretation

## 6.1 Main Outcomes

- DeepPoly is consistently more precise than compared scalable baselines in most settings.
- DeepPoly scales to large feedforward and convolutional networks.
- DeepPoly proves robustness for rotation+interpolation setting when paired with refinement.

## 6.2 Performance Trends

- On feedforward networks, DeepPoly often outperforms DeepZ in both proofs and speed.
- On some convolutional settings, DeepPoly proves more but can be slower.
- Difference widens at larger epsilon values in many cases (where precision pressure is higher).

## 6.3 Failure and Edge Cases

- Without refinement, complex geometric perturbations may fail certification.
- Coarse abstraction can fail even when model may actually be robust.
- Sparse convolutional affine structures may favor alternative abstract representations for speed.

## 6.4 Statistical Meaning (Beyond Raw Numbers)

- The paper emphasizes consistent directional improvements across model families.
- Evidence is empirical-comparative, not inferential-statistical.
- Interpretation should be: strong engineering and algorithmic signal, but not a full statistical generalization study.

### Publishability Strength Check

- Publication-grade strengths:
- new sound domain design with proofs
- practical implementation plus extensive benchmark comparisons
- first demonstration for this robustness style under rotation+interpolation
- Aspects needing stronger validation in follow-up work:
- larger and more diverse test pools
- stronger statistical treatment
- more ablation studies quantifying each design choice separately

# 7. Strengths - Weaknesses - Assumptions

## 7.1 Technical Strengths

| Strength | Why It Matters |
|---|---|
| Custom domain tailored to neural operations | Better precision-speed trade-off than generic domains |
| Soundness theorems plus invariant preservation | Gives trustworthy certification claims |
| Floating-point soundness handling | Closer to deployment reality |
| Supports feedforward and convolutional architectures | Broad practical applicability |
| Refinement strategy for rotation with interpolation | Extends beyond standard L-infinity-only scenarios |

## 7.2 Explicit Weaknesses

| Weakness | Practical Effect |
|---|---|
| Over-approximation remains conservative | Some robust inputs cannot be proved |
| Runtime can be high on large convolutional cases | Practical throughput limitations |
| Rotation verification needs heavy partitioning in hard cases | Potential computational overhead |
| Evaluation subset size is limited | External validity is not fully established |

## 7.3 Hidden Assumptions

| Hidden Assumption | Risk |
|---|---|
| Interval perturbations are representative of threats | Real-world corruptions may differ structurally |
| Correctly classified test points are sufficient anchor set | Misclassification distribution may bias robustness picture |
| Verification runtime remains acceptable in deployment loops | May break for stricter latency constraints |
| Refinement granularity can be tuned manually | Manual tuning may reduce reproducibility at scale |

# 8. Weakness to Research Direction Mapping

| Identified Weakness | Why It Exists | Research Opportunity | Possible Method |
|---|---|---|---|
| Conservative abstraction causes proof failures | Linearized bounds lose detail | Precision-adaptive certification | Dynamic split of only high-impact neurons/layers |
| Slowdowns on large convolutional models | Bound propagation not always sparsity-optimal | Hybrid abstract domains | Combine DeepPoly with sparse zonotope or LP relaxations |
| Heavy refinement for rotations | Global geometric over-approximation is loose | Geometry-aware abstraction | Piecewise differentiable transform abstractions with adaptive partitions |
| Limited statistical reporting | Verification papers prioritize solver comparisons | Stronger empirical methodology | Multi-seed, confidence intervals, and robust aggregate metrics |
| One-threat-family focus per experiment | Benchmark convention | Threat-composition certification | Joint certification for blur, shift, rotation, and pixel noise |

# 9. Novel Contribution Extraction

We propose a neural-network verification domain that improves certification precision by combining relational linear bounds and interval bounds with operation-specific abstract transformers.

We propose a floating-point-sound certification workflow that preserves formal guarantees under practical numerical computation.

We propose a refinement-assisted verification strategy for rotation with interpolation, enabling certification for perturbations not handled by plain box propagation.

## 3-5 Reusable Novel Claim Templates

1. We propose ______, a verification abstraction that improves certified robustness rate by ______ while preserving soundness under floating-point arithmetic.
2. We introduce ______ transformers that reduce approximation error for ______ operations and improve proof success at larger perturbation budgets.
3. We develop a refinement mechanism for ______ perturbations that enables certification where single-pass abstractions fail.
4. We show that combining ______ and ______ yields a better precision-runtime frontier than prior scalable analyzers.
5. We provide evidence that ______ architecture classes can be verified at scale with ______ without sacrificing formal correctness.

# 10. Future Research Expansion Map

## 10.1 Author-Suggested or Implied Directions

- Use improved transformers during adversarially robust training.
- Extend practical certification coverage while retaining scalability.

## 10.2 Missing Directions

- Certification for richer non-linear image pipelines (camera ISP effects, weather-like transformations).
- Certified robustness under distribution shift, not only norm-bounded perturbation.
- Better compositional verification for end-to-end perception stacks.

## 10.3 Modern Extensions (LLM Era and Emerging)

- Use large-model-guided heuristic selection for refinement partitions.
- Neuro-symbolic coupling: LLM-generated candidate invariants for proof search.
- Hardware-aware certified inference constraints (quantization and mixed precision).
- Unified certification benchmark suites with reproducible pipelines and auto-reporting.

## 10.4 Cross-Domain Combination Ideas

- Robust perception in autonomous driving: combine geometric and photometric certified perturbations.
- Medical imaging: certify diagnosis model invariance under clinically plausible acquisition variation.
- Security pipelines: certify downstream decision modules jointly with perception nets.

# 11. How to Write a New Paper From This Work

## 11.1 Reusable Elements

- Problem framing: precision vs scalability tension in certifiable robustness.
- Method structure: define abstraction, prove soundness, design tailored transformers, then evaluate at scale.
- Evaluation style: report proved-robust percentage and runtime across architectures and threat sizes.

## 11.2 What Must Not Be Copied

- Do not reuse identical transformer formulas without a new technical extension.
- Do not reuse benchmark settings unchanged as sole novelty.
- Do not present same threat model and same architecture families without a distinct contribution.

## 11.3 How to Design a Novel Extension

1. Pick a concrete weakness from Section 8.
2. Define one new abstraction or transformer component that directly targets that weakness.
3. Prove at least soundness and invariant preservation (or equivalent formal property).
4. Add ablations showing contribution of each method part.
5. Evaluate against strongest relevant baselines, not only older ones.

## 11.4 Minimum Publishable Contribution Checklist

- Clear technical novelty beyond DeepPoly baseline.
- Formal correctness argument.
- Implementation with reproducible artifacts.
- Strong comparative evaluation on diverse models.
- Honest failure analysis and limitations.

# 12. Publication Strategy Guide

## 12.1 Suitable Venue Types

- Programming languages and verification venues (formal methods + systems).
- Security and trustworthy AI venues.
- Top ML venues if empirical and method novelty are both strong.

## 12.2 Baseline Expectations

- Compare with current strong certifiers, not only historical baselines.
- Include both certified robustness and runtime/memory trade-offs.
- Include defended and undefended models.

## 12.3 Experimental Rigor Level Needed

- Multiple datasets and architecture scales.
- Clear hardware/software configuration.
- Deterministic or seeded reproducibility details.
- Ablation and sensitivity analyses.

## 12.4 Common Rejection Reasons

- Incremental extension without formal depth.
- Evaluation too narrow or unfair.
- Missing comparisons to strongest recent methods.
- No clear explanation of where precision/runtime gains come from.

## 12.5 Increment Needed for Acceptance

- Strong: new abstraction/theory + robust implementation + convincing large-scale evidence.
- Moderate: focused domain extension with clear formal and empirical gains.
- Weak: only engineering speedup without proof or generality.

# 13. Researcher Quick Reference Tables

## 13.1 Key Terminology Table

| Term | Simple Meaning | Use in Paper |
|---|---|---|
| Adversarial region | Allowed perturbation set | Input set to certify |
| Abstract element | Compact set representation | Tracks all possible activations |
| Transformer | Operation-wise updater | Propagates abstraction through network |
| Backsubstitution | Replace intermediate symbols recursively | Tightens concrete bounds |
| Trace partitioning | Split analysis into cases | Refinement for complex transforms |

## 13.2 Important Equations Summary

| Equation/Concept | Purpose |
|---|---|
| Concretization gamma(a) | Maps abstract element to concrete set |
| ReLU crossing-zero linear bounds | Safe approximation where ReLU is ambiguous |
| Affine transformer substitution formulas | Compute tighter l_i and u_i |
| Output margin constraints | Convert class robustness to positivity checks |
| Complexity O(n_max^2 * L) (per variable bound step) | Runtime intuition for affine bound computation |

## 13.3 Parameter Meaning Table

| Parameter | Meaning |
|---|---|
| epsilon | L-infinity perturbation radius |
| alpha, beta | Rotation angle range |
| n_batches | Number of angle partitions |
| batch_size | Number of interval refinements per partition |
| l_i, u_i | Concrete bounds per neuron |

## 13.4 Algorithm Flow Summary

| Stage | Input | Output |
|---|---|---|
| Input abstraction | Perturbation spec | Initial abstract state |
| Layer propagation | Abstract state + assignments | Final abstract output state |
| Margin construction | Output activations | Class-difference variables |
| Decision | Margin lower bounds | Proved/Not proved |
| Optional refinement | Failed proof + complex transform | Partitioned re-analysis |

# 14. One-Page Master Summary Card

## Problem

- Neural networks are vulnerable to tiny and structured perturbations.
- Need formal, scalable, floating-point-sound robustness certification.

## Idea

- Use DeepPoly: relational linear bounds plus interval bounds with tailored transformers.

## Method

- Translate network to assignment sequence.
- Propagate abstract state with operation-specific transformers.
- Use recursive backsubstitution for tighter affine bounds.
- Check class margins via positivity of derived lower bounds.
- Apply refinement (trace partitioning) for difficult geometric perturbations.

## Results

- More proved properties than strong scalable baselines in most tested settings.
- Scales to large feedforward and convolutional models.
- Demonstrates robustness proofs under rotation with interpolation via refinement.

## Weakness

- Conservative approximations still miss some truly robust cases.
- Runtime can be higher on large convolutional networks.

## Research Opportunity

- Adaptive hybrid abstractions and geometry-aware refinement to close precision-speed gaps.

## Publishable Extension Blueprint

- Introduce one targeted abstraction improvement.
- Prove soundness/invariant preservation.
- Demonstrate gains on modern certified robustness benchmarks with strong baselines.
