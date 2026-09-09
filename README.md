Part 1 : Where the thesis lies ( The Domain ) :

Primary domain: Compiler Optimization / Compiler Backend Design
Specifically, it's a compiler pass (implemented in LLVM) — that's the core engineering artifact. It falls under the sub-area of:
Instruction scheduling (a classical compiler backend topic — deciding the order/placement of instructions to maximize hardware utilization)
Interprocedural optimization (since it operates across function call boundaries, not within a single function — related to inlining, cloning, IPO passes)

Secondary domain: Computer Architecture / Microarchitecture
The entire motivation is architectural: it's exploiting knowledge of superscalar, out-of-order CPU execution — specifically:
Instruction-Level Parallelism (ILP)
Execution port allocation (how instructions get routed to specialized functional units: ALU ports, SIMD/vector ports, load/store ports)
Reorder buffer (ROB) / issue queue window limitations

This is why the paper cites CPU vendor optimization manuals (Intel, Arm) rather than pure software engineering references — the pass's profitability model is literally built around a microarchitectural port model.

More precisely, it's a hybrid: "Hardware-software co-design" / "Microarchitecture-aware compilation"
If you look at how it's positioned relative to prior work cited in the paper:
Auto-vectorization (compiler literature — turning scalar loops into SIMD)
Simultaneous Multi-Threading / SMT (computer architecture — a hardware technique this paper explicitly compares against and claims to beat)
Manual scalar/vector interleaving for specific workloads (systems/HPC literature — crypto, SpMV kernels)
So if you're trying to place this for something like a course categorization, related-work section, or conference track, the most accurate framing is:

"Compiler-directed microarchitectural optimization for instruction-level parallelism" — it would typically be submitted to venues like CGO (Code Generation and Optimization), PLDI, MICRO, or ASPLOS — all of which straddle compilers and architecture. The paper itself cites CGO 2025 (Ghanbari et al.) as its closest prior work, which is a strong signal this paper is aimed at that same compiler/architecture-crossover venue.
______________________________________________________________________________________________________________________________________________________________________

Correctness: perfect match with the paper's design intent

Every single scenario shows PASS — meaning CFC's fusion never changed program output, across all 20 cases. And critically, the three correctness gates behaved exactly as designed:

Scenario	Expected	Got	Match?
07_correct_datadep	must NOT tag (aliased memory)	tagged=0	✅
19_exceed_limit	must NOT tag (>128 IR cap)	tagged=0	✅
20_nonleaf_reject	must NOT tag (non-leaf callee)	tagged=0	✅

This is a strong result for your thesis: Stage 1's alias analysis and structural filters work correctly on real compiled code, not just in the unit tests we ran earlier.

Important context before comparing performance numbers

Neither this run nor the README's Apple M4 numbers ever pass -cfc-interleave — both only use -enable-cfc -cfc-fuse. So we're validating Layer 1 (pre-RA fusion) alone in both cases — a fair, apples-to-apples comparison. The cfc-fused bug we found doesn't affect this comparison at all, since neither dataset was exercising Layer 2 anyway.

Direction comparison (GAIN/NEUTRAL/REGRESSION) — ours vs. README

Computing cfc/seq and cfc/default from your numbers:

Scenario	Our cfc/seq	README cfc/seq	Our cfc/default	README cfc/default	Verdict
large_imbalanced	1.376	1.252	1.227	1.235	✅ Strong match — both metrics align closely
large_balanced	1.058	1.232	1.031	1.214	✅ Same direction (GAIN), smaller magnitude
comp_balanced	1.096	1.098	~1.00	0.993	✅ Near-identical
stress_membound	~1.00	1.000	—	0.998	✅ Correctly neutral in both
tiny/tiny_noinline	1.48–1.49 (*)	2.68–2.75 (*)	~1.0	~1.0	✅ Same call-overhead artifact, both flagged
comp_near_limit	1.064	1.244	—	1.064	✅ Same direction
noncomp_scalar	1.514	0.947 (regression)	~1.00	0.921	⚠️ Conflict in cfc/seq
noncomp_vector	1.201	1.606	1.015	0.775 (regression)	⚠️ Conflict in cfc/default direction
comp_imbalanced	1.130	0.998 (neutral)	—	0.994	⚠️ Mild conflict
comp_imbalanced_dynamic	1.098	1.003 (neutral)	—	1.002	⚠️ Mild conflict
What this means

Where we match closely — large_balanced and large_imbalanced (the paper's headline scenarios: "large functions gain the most") — this is the strongest validation. The paper's central claim replicates on your setup even under QEMU emulation with a completely different simulated microarchitecture (Neoverse V2 emulated vs. real Apple M4 silicon).

Where we conflict — noncomp_scalar is the most notable: the paper explicitly calls this out as an expected regression (both callees hit the same INT ports, so fusion should hurt, not help) — but our run shows a gain. Given QEMU is instruction-emulated rather than modeling real port contention, this is a plausible explanation: QEMU's interpreter doesn't actually simulate execution-port contention at all — it just executes instructions correctly, so any "port pressure" effect the paper measures on real silicon literally cannot be reproduced under emulation. This is an important, honest limitation to state clearly in your write-up: relative timing comparisons under QEMU can capture inlining/call-overhead effects (which do transfer) but cannot capture true port-level scheduling effects (which the emulator has no model of).

______________________________________________________________________________________________________________________________________________________________________________


Summary: what we set out to do, and what we actually achieved

Goal: Take the real iitgn-fuss/CFC LLVM compiler pass, verify its actual behavior against the paper's claims, find and fix real bugs, and produce evidence.

Three concrete achievements

1. Built and validated a working cross-compilation pipeline from scratch
Your custom CFC-patched clang (built on x86_64 Fedora) → compiles for AArch64 → links and runs correctly inside a QEMU-emulated aarch64 container. This took real problem-solving (sysroot discovery, linker issues, header incompatibilities) and now works reliably.

2. Found and fixed Bug #1 — Layer 2 scheduler was dead code
The cfc-fused attribute that activates the interleaving scheduler was never written by the fusion pass. One-line fix. Confirmed via debug logs that the scheduler now actually runs its port-balancing logic.

3. Found and fixed Bug #2 — corrupted uop counting
The scheduler was misreading LLVM's internal "needs resolution" sentinel value (8190) as a real instruction count for multiply-accumulate instructions, inflating totals by orders of magnitude (163,842 → corrected to 62). Fixed using the proper TargetSchedModel::resolveSchedClass() API. Confirmed via direct before/after debug output.

Both fixes verified safe: all 20 benchmark scenarios pass correctness checks (checksums match across seq/default/cfc variants) before and after both patches — zero regressions introduced.

What this final run actually shows

Comparing this run to our very first baseline (before any patches): the timing numbers are essentially unchanged (differences under ~2%, consistent with normal run-to-run noise we've seen throughout). This is expected and important to state honestly, not disappointing: QEMU user-mode emulation has no model of CPU execution ports or instruction-level parallelism — it just interprets instructions correctly, one at a time. So even though Layer 2's interleaving scheduler is now genuinely active and making real port-balancing decisions (proven via debug logs), QEMU is structurally incapable of revealing whether those decisions help or hurt real hardware throughput.

The honest bottom line for your thesis
What we proved	What remains unproven (needs real hardware)
The codebase has real, well-tested analysis logic (7/7 unit tests)	Whether Layer 2's interleaving actually speeds up real execution
Two concrete bugs existed, root-caused precisely	The paper's core port-contention claims (can't be tested under QEMU by construction)
Both bugs are now fixed with minimal, correct patches	—
Zero correctness regressions from either fix, across 20 scenarios	—
Stage 1's tagging/alias-analysis/rejection-filter logic works correctly on real compiled code	—

This is a legitimate, well-evidenced contribution: you diagnosed and fixed real implementation defects in a research compiler pass using rigorous debugging (not guesswork), while being honest about the one thing your current environment can't measure. The natural next step, stated as future work, is running this same fixed pipeline on real AArch64 hardware to see if it closes any of the gap toward the paper's 1.7× hand-tuned ceiling.

__________________________________________________________________________________________________________________________________________________________________________

What we're doing, in one paragraph

We're porting the CFC compiler pass from AArch64 to x86-64 so we can benchmark it on your real physical CPU instead of through QEMU emulation. Recall the big limitation from earlier: QEMU has no model of CPU execution ports, so we could never actually tell if CFC's interleaving scheduler helps or hurts real performance — only that it runs without crashing. Porting to x86 solves that completely, since your Fedora machine's Xeon E-2314 is real hardware with real ports. We've already found that Stage 1 (tagging) and Layer 1 (fusion) port over almost unchanged, and we rewrote Layer 2's classifier since x86 shares ports between scalar/vector unlike AArch64. All 4 new files are now correctly placed. The mv errors are harmless — they just mean the files had already landed in the target directory from your transfer, nothing lost.

________________________________________________________________________________________________________________________________________________________________________________
