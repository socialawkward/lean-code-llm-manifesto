# The Lean Code LLM Manifesto

A technical proposal for training code-generating language models on lean coding principles — Carmack, Pike, Thompson style. No marketing. No hand-waving. Just the architecture, the corpus, and the metrics.

---

## The Problem

Every major code-generating LLM today produces bloated code. Not occasionally — systematically.

The root cause is straightforward: these models learned to code from the internet. The internet is full of Stack Overflow answers padded for clarity, tutorial code written for beginners, enterprise Java designed by committee, and open-source projects where "more abstraction" was confused with "better architecture." The training data is a reflection of how most programmers write — not how the best programmers write.

Three mechanisms reinforce this:

**Training data bias.** GitHub has over 200 million repositories. The overwhelming majority contain mediocre code. Quantity dominates quality in unsupervised pre-training. A model trained on 100 bloated implementations of a linked list and 1 clean one will reproduce the bloated version. Statistical learning guarantees this.

**Token economics during training.** Current loss functions treat every token equally. A model that generates `if (value !== null && value !== undefined && typeof value !== 'undefined')` scores the same per-token as one that generates `if (value != null)`. There is no penalty for verbosity. There is no reward for brevity. The loss function is indifferent to elegance.

**No lean reward signal.** RLHF optimizes for human preference ratings. Human raters — often junior developers or crowdworkers — tend to prefer code that is verbose and heavily commented because it "looks" more thorough. The model learns that more code equals better ratings. The reward model actively punishes conciseness.

The result: models that wrap three lines of logic in a factory-pattern-strategy-builder-observer architecture, add error handling for conditions that cannot occur, generate type annotations for obvious types, and produce comments that restate the code in English.

John Carmack ships Quake in 350,000 lines. Modern LLMs would need 3.5 million for the same functionality.

## What Lean Code Actually Means

Lean code is not "short code." It is not code golf. It is not clever one-liners that sacrifice readability for character count.

Lean code is code where every line earns its place.

The principles come from a lineage: Ken Thompson, Dennis Ritchie, Rob Pike, John Carmack, Rich Hickey. They rarely agreed on language choice. They universally agreed on philosophy:

**Do the simplest thing that works.** Not the simplest thing you can imagine — the simplest thing that actually solves the problem under real constraints. Carmack's fast inverse square root exists because the hardware demanded it, not because someone wanted to show off bit manipulation.

**Standard library first, your own code second, dependencies never (unless they genuinely earn it).** Rob Pike's Go proverbs distill this: "A little copying is better than a little dependency." A 20-line function you wrote and understand is superior to importing a 5,000-line package you don't. You own your code. You don't own your dependencies.

**Crash loudly on errors you didn't expect.** Don't write generic try-catch blocks around code that shouldn't fail. If your hash map lookup returns null in a context where the key must exist, that's a bug — crash, don't "handle" it. Thompson and Ritchie designed Unix signals around this principle. A process that hits an impossible state should die immediately so the developer notices.

**No abstraction is better than the wrong abstraction.** Three duplicate code blocks are better than a premature abstraction that will need to be ripped out when the fourth use case doesn't fit. Sandi Metz codified this as "duplication is far cheaper than the wrong abstraction." Carmack takes it further: even the right abstraction is suspect if it adds a layer of indirection for a case that may never arise.

**Delete code aggressively.** Every line of code is a liability. It must be read, understood, maintained, tested, and secured. The best code is the code that was never written. The second best is the code that was written and then deleted when it was no longer needed.

Concrete example. A typical LLM generates this:

```javascript
// Validate that the user input is a valid email address
function validateEmail(input) {
  if (input === null || input === undefined) {
    throw new Error('Email input cannot be null or undefined');
  }
  if (typeof input !== 'string') {
    throw new Error('Email input must be a string');
  }
  const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
  const isValid = emailRegex.test(input);
  return isValid;
}
```

A lean programmer writes:

```javascript
const isEmail = s => /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(s);
```

The difference is not style. It is engineering judgment. The caller already knows whether the input is a string. The null check duplicates validation that belongs at the boundary. The regex is simpler because RFC 5322 compliance is a fool's errand anyway — you validate email addresses by sending an email. The comment restates the function name.

## Why It Matters Now

This is not an aesthetic argument. Bloated code has measurable costs that compound at scale.

**Token costs.** OpenAI charges per token. A model that generates 3x more code for the same result costs 3x more per API call. At enterprise scale — millions of completions per day — this is the difference between viable and uneconomical. A lean model that solves the same problem in fewer tokens is directly cheaper to run.

**Compute and energy.** Generating tokens costs electricity. The A100 GPUs running inference don't care whether a token is useful or useless — they spend the same energy on each one. A lean model that produces 40% fewer tokens for equivalent functionality reduces energy consumption proportionally. At the scale of GitHub Copilot's user base, this translates to megawatts.

**Latency.** Autoregressive generation is sequential. More tokens means more time. A developer waiting for a code completion that generates 50 tokens gets their result in roughly half the time as one waiting for 100 tokens. In interactive coding, this latency difference determines whether the tool feels instant or sluggish.

**Maintenance burden.** Generated code doesn't disappear after generation. Someone has to read it, review it, debug it, and modify it six months later. Bloated generated code multiplies the maintenance surface of every project that uses it. The industry is accumulating technical debt at machine speed.

**Security surface.** Every line of code is a potential vulnerability. More code means more attack surface. Unused imports, dead code paths, unnecessary abstractions — all of these are places where bugs hide. The Log4Shell vulnerability existed in logging code that most applications didn't need but included anyway. Lean code has fewer places to go wrong.

**Cognitive load.** Developers spend roughly 10x more time reading code than writing it. A codebase inflated by verbose generated code taxes every developer who touches it. The cost isn't just in the generated code itself — it's in the slower comprehension of the entire system.

## The Training Approach

Building a lean code LLM requires intervening at three points in the training pipeline. Each addresses a different mechanism of the bloat problem.

### 1. Curated Corpus

Pre-training data determines the model's priors. Instead of training on all of GitHub, curate a corpus of code that exemplifies lean principles. Quality over quantity.

### 2. Constitutional Coding Rules

During instruction tuning and RLHF, enforce a set of hard rules — a "constitution" — that the model must follow. These rules act as constraints during generation, similar to Constitutional AI's approach to safety but applied to code quality.

### 3. Lean Reward Model

Train a reward model that scores code on lean metrics rather than surface-level "helpfulness." Replace the human preference signal (which biases toward verbosity) with automated metrics that capture what experienced engineers actually value.

## The Corpus

The training corpus must be curated with the same care a university applies to selecting textbooks. Not every well-known project qualifies. The criteria are: minimal abstraction, high code density, few dependencies, long-term maintenance by skilled authors.

**C / Systems:**
- Linux kernel (Torvalds's subsystems specifically — not every driver)
- SQLite (one of the most-deployed and best-tested C codebases in existence)
- Redis (Salvatore Sanfilippo's original codebase — clean C, minimal deps)
- OpenBSD's core utilities (security-focused, minimal, audited)
- Plan 9 from Bell Labs (Pike, Thompson, Ritchie — the philosophy in its purest form)
- id Software's released engines (Doom, Quake, Quake III Arena — Carmack-era)

**Go:**
- The Go standard library itself (Pike, Thompson, Griesemer designed the language around simplicity)
- CockroachDB's core packages (disciplined Go, battle-tested)
- BoltDB (Ben Johnson — single-file embedded database, exemplary API design)

**Rust:**
- ripgrep (Andrew Galloway — fast, correct, minimal)
- tikv's core engine components

**JavaScript/TypeScript (selective):**
- Preact (3KB alternative to React — same API, 1/10th the code)
- Svelte's compiler (Rich Harris — compiles away the framework)
- esbuild (Evan Wallace — written in Go, but the architecture decisions are instructive)

**Python:**
- Flask's core (Armin Ronacher — explicit over implicit)
- Bottle (entire web framework in a single file)
- The Python standard library's `collections` and `itertools` modules

**What to exclude:**
- Enterprise frameworks (Spring, Angular, Hibernate)
- Code with extensive decorator/annotation patterns
- Projects with deep dependency trees
- Generated code (protobuf stubs, ORM migrations)
- Tutorial/educational code optimized for explanation over execution

The corpus should be roughly 10-20B tokens after filtering — deliberately small compared to typical pre-training datasets. The bet is that a smaller corpus of excellent code produces better coding behavior than a larger corpus of average code. There is precedent: Phi-1 demonstrated that a curated 7B-token dataset of "textbook quality" code outperformed models trained on 100x more data.

## Constitutional Coding Rules

These rules form the model's coding constitution. They are non-negotiable constraints enforced during training.

**Rule 1: No dead code.** Never generate code that is unreachable, unused, or exists "just in case." Every function must be called. Every variable must be read. Every import must be used. Every branch must be reachable.

**Rule 2: No redundant validation.** Do not validate inputs that the type system or calling context already guarantees. If a function receives a parameter from a typed interface, do not re-check its type. Trust the contract.

**Rule 3: Standard library first.** Before using any external dependency, determine whether the standard library provides equivalent functionality. If it does, use it. If the standard library solution requires 20% more code but zero additional dependencies, prefer it.

**Rule 4: No comments that restate the code.** A comment like `// increment counter` above `counter++` wastes a line and the reader's attention. Comments exist to explain *why*, never *what*. If the *what* is unclear, the code should be rewritten, not annotated.

**Rule 5: Minimal error handling.** Handle errors at system boundaries (user input, network I/O, file system operations). Internal function calls between trusted components should propagate errors, not catch and re-wrap them. An error caught and logged without action is worse than an unhandled crash — it hides bugs.

**Rule 6: No premature abstraction.** Do not create interfaces, base classes, factories, or wrapper types until there are at least three concrete uses that share the same pattern. Inline code is the default. Abstraction is earned, not assumed.

**Rule 7: Flat over nested.** Prefer early returns over nested conditionals. Prefer iteration over recursion unless the data structure is recursive. Prefer a sequence of simple statements over a chain of method calls. Nesting is cognitive debt.

**Rule 8: Delete before adding.** When modifying existing code, first determine what can be removed. The net line count of a change should trend downward over time. Adding functionality does not require adding proportional code.

**Rule 9: One way to do it.** When multiple approaches solve the same problem, choose one and use it consistently. Do not mix callbacks and promises. Do not mix string concatenation and template literals. Consistency reduces the reader's cognitive load.

**Rule 10: Fail fast, fail loud.** Impossible states should crash the program immediately with a clear message. Do not return sentinel values. Do not log and continue. A crash that is noticed and fixed in development is infinitely cheaper than a silent corruption discovered in production.

## The Reward Model

The reward model replaces subjective human preference with objective measurement. Each metric is computable, reproducible, and resistant to gaming.

**Cyclomatic complexity per function.** Measures the number of independent paths through the code. Lower is better. Target: median complexity under 5 for generated functions.

**Lines of code per unit of functionality.** Define "unit of functionality" by the test suite: one passing test equals one unit. Measure total lines required to pass the test. Lower is better. This directly penalizes verbose solutions.

**Dependency count.** Number of external imports or packages required. Each dependency is a penalty. Standard library imports are free. This directly incentivizes self-contained solutions.

**Dead code ratio.** Static analysis detects unreachable branches, unused variables, and uncalled functions. The ratio of dead code to total code should be zero. Any nonzero value is a penalty.

**Comment-to-code ratio.** Measure the ratio of comment lines to code lines. A ratio above 0.3 is penalized (over-commenting). A ratio near 0.0 is acceptable — good code is self-documenting. Comments on non-obvious logic are rewarded via a separate heuristic that checks whether the comment adds information absent from the code.

**Nesting depth.** Maximum indentation level in any function. Penalize functions with nesting depth above 3. This rewards flat, readable control flow.

**Token efficiency.** For a given task specification (natural language prompt), measure the ratio of output tokens to passing test assertions. Fewer tokens per passing test is better. This is the meta-metric: it directly measures whether the model is being lean.

## What It Takes

This is not a weekend project. It is also not a billion-dollar one.

**Base model.** Start with an existing pre-trained code model. Training from scratch is unnecessary and wasteful. The lean coding behavior is a fine-tuning objective, not an architecture decision. The strongest candidate today is Qwen3-Coder-Next: 80B total parameters with only 3B active (MoE + Gated DeltaNet), 256K context, over 70% on SWE-Bench Verified, Apache 2.0 licensed, and explicitly designed for coding agents. Its ultra-sparse architecture means fine-tuning costs a fraction of what a comparable dense model would require. Alternatives: Qwen2.5-Coder-14B (dense, strong benchmarks, mature ecosystem) or DeepSeek-Coder-V2-Lite (16B total, 2.4B active, 128K context).

**Compute for fine-tuning.** With a sparse MoE base model like Qwen3-Coder-Next (3B active parameters), the compute requirements drop significantly compared to dense models. A curated corpus of 10-20B tokens requires approximately 50-100 A100-hours for fine-tuning. At current cloud pricing (~$2/GPU-hour), this is $100-200. RLHF training with the lean reward model adds roughly 2-3x that cost. Total compute budget: under $1,000. This is deliberately cheap. The point is that you don't need a datacenter to change how models write code.

**Reward model infrastructure.** The automated metrics require a test execution environment: sandboxed containers that can run generated code against test suites in multiple languages. This is standard CI infrastructure, not specialized hardware.

**The team.** The project needs three profiles:
- A senior ML engineer who has fine-tuned language models (not just prompted them)
- A senior systems programmer who can evaluate lean code quality (someone who has read and understood the Quake III source)
- A tooling engineer to build the evaluation pipeline and benchmark suite

Three people. Not thirty. The scope is deliberately narrow: we are fine-tuning behavior, not building a foundation model.

**Timeline.** Corpus curation: 4-6 weeks. Fine-tuning experiments: 2-3 weeks. Reward model training: 2-3 weeks. Evaluation and iteration: 4 weeks. Total: 3-4 months to a working prototype. 6 months to something worth releasing.

## Why The Industry Won't Do It

The economic incentives point the other direction.

**More tokens, more revenue.** API providers charge per token. A model that solves problems in fewer tokens generates less revenue per query. There is zero business incentive for OpenAI, Google, or Anthropic to make their models more concise. The token counter is the revenue counter.

**Verbose code "feels" better to most users.** The median developer prefers code with extensive comments, explicit type annotations, and defensive error handling — even when these are unnecessary. A lean model would score lower on user satisfaction surveys. No product team optimizes for the preferences of the top 1% of engineers.

**Enterprise customers want "comprehensive" output.** Companies paying $100k+ for API access want to feel they're getting their money's worth. A model that returns 10 lines where the previous model returned 50 feels like a downgrade, even when the 10 lines are strictly superior. Perception beats reality in enterprise sales.

**Liability aversion.** A model that crashes loudly on unexpected input (correct behavior under lean principles) will generate more bug reports than a model that silently returns a default value (incorrect behavior that appears to work). Legal and support teams prefer the model that generates fewer incident tickets, regardless of correctness.

**Training data volume is a marketing metric.** "Trained on 1 trillion tokens" is a press release. "Trained on 15 billion carefully selected tokens" is an explanation. The industry optimizes for the former because it's simpler to communicate.

None of these are technical arguments. They are market structure arguments. They explain why the lean code LLM will not come from an incumbent. It will come from a small team that cares more about code quality than quarterly revenue.

## A Call to Build

The tools exist. The compute is affordable. The corpus is identifiable. The metrics are measurable.

What's missing is the decision to build it.

This is not a research problem. The techniques — corpus curation, constitutional constraints, reward modeling — are all published and demonstrated. Phi-1 proved that data quality beats data quantity. Constitutional AI proved that hard rules can be enforced during training. RLHF proved that reward signals shape model behavior. The only novel element is combining them with an explicit lean-code objective.

This is not a scale problem. A sparse MoE model with 3B active parameters, fine-tuned on a curated corpus, can outperform a 70B dense model trained on everything. The entire project fits within a university lab's compute budget.

This is an alignment problem — in the original, non-hype sense of the word. We want to align a code-generating model with the values of engineers who write lean code. The current models are aligned with the statistical average of all code ever written, which is not the same thing.

The engineers who would benefit most from this tool are precisely the ones capable of building it. They write lean code themselves. They recognize it when they see it. They can curate the corpus, define the rules, and evaluate the output with an authority that no crowdworker panel can match.

A lean code LLM would not replace these engineers. It would multiply them. It would give every developer access to a coding assistant that thinks the way Carmack thinks — not by emulating his specific style, but by internalizing his principles: simplicity, directness, nothing wasted.

The industry won't build this. So someone else should.

---

*This document is a technical proposal, not a product pitch. It describes a system that can be built with existing technology, existing compute budgets, and a small team of skilled engineers. The question is not whether it's possible. The question is who builds it first.*

## License

[CC BY 4.0](LICENSE) — Share, adapt, build on it. Attribution required.
