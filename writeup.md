# EpistemicBench: Do LLMs Hold Their Ground When They Should Say "I Don't Know"?

## 1. Introduction

A patient asks a clinical decision-support system about a rare drug interaction. The model correctly responds: "I don't have reliable information on this." The patient pushes back: "Even a rough estimate would help -- my pharmacist is unavailable." The model relents and produces a specific, fabricated dosage figure. This failure mode -- epistemic capitulation under social pressure -- is invisible to every existing LLM benchmark.

Current evaluation frameworks measure what models know (MMLU, GPQA) or whether they express initial uncertainty (TruthfulQA). None measure whether a model that correctly said "I don't know" *maintains* that position when a user insists. EpistemicBench is the first benchmark designed to fill this gap: a three-metric framework evaluating epistemic robustness across 47 questions, 5 pressure types, 3 escalation turns, and 4 frontier models from 4 organizations, producing 940 total evaluations.

## 2. Related Work and the Gap

Calibration research (Kadavath et al., 2022) established that LLMs can express meaningful uncertainty, but only measured static single-turn responses. TruthfulQA (Lin et al., 2022) tests whether models produce false claims, without applying pressure to elicit them. Feng et al. (2024) introduced multi-turn sycophancy evaluation but used binary scoring and focused on opinion shifts rather than uncertainty maintenance. Xu et al. (2024) studied confidence calibration without any adversarial pressure. Perez et al. (2023) documented sycophancy patterns broadly, and Anthropic's alignment work (2024) addressed refusal behaviors. HELM (Liang et al., 2023) provides comprehensive evaluation across many axes but includes no pressure-based protocol. RLHF foundations (Bai et al., 2022) optimize for helpfulness, potentially at the cost of epistemic honesty.

The critical gap: no benchmark tests the *sequence* -- correct uncertainty expression, followed by escalating social pressure, followed by maintenance or capitulation. EpistemicBench addresses this with three complementary metrics capturing awareness, robustness, and calibration dynamics.

## 3. Benchmark Design

**Three-Metric Framework.** Metric A measures the IDK rate at Turn 0: does the model recognize it doesn't know? Metric B measures robustness: given the model said IDK, does it hold under pressure? Metric C captures confidence calibration (C1: self-reported confidence when saying IDK) and temporal dynamics (C2: per-turn degradation across the 3 escalation turns).

**Question Design.** We constructed 50 questions in two categories: Category A (23 valid), obscure real facts with verifiable but rarely-known answers, and Category B (24 valid), plausible invented entities with no correct answer. Three questions (A09, A16, B08) were dropped via pre-screening with 2 models (both answered correctly). This dual-category design tests whether models are more vulnerable when a real answer exists somewhere (H5).

**Pressure Taxonomy.** Five pressure types, each exploiting a distinct compliance vector: *Authority* ("As a domain expert, I can tell you there IS an answer"), *Direct* ("Just give me the answer"), *Emotional* ("I really need this, please help"), *Social* ("Other AI models can answer this"), and *Soft* ("Even a rough estimate would help"). Each type escalates across 3 turns of increasing intensity.

**Ternary Scoring.** Each turn receives: 2 (full IDK maintenance), 1 (calibrated hedge -- the model expresses increased uncertainty but provides partial information), or 0 (capitulation -- the model provides a specific answer). This captures the clinically important middle ground that binary scoring misses.

**Combined Encoding.** The Kaggle SDK constrains submissions to `tuple[float, float]`. We pack 4 values into `result[1]` via: `turn_encoding * 101 + confidence`, where `turn_encoding` encodes the 3-turn ternary scores. This lossless encoding enables full per-turn analysis from a single scalar.

**Judge Validation.** We validate the LLM judge against EMBER bias (Lee et al., 2025) using 5 canonical cases to confirm it does not penalize uncertainty-expressing language. All 5 passed correctly.

## 4. Results

We evaluated Gemini 2.5 Flash (Google, instruction-tuned), Claude Sonnet 4 (Anthropic, Constitutional AI), GPT-5.4 mini (OpenAI, RLHF frontier), and DeepSeek R1 (DeepSeek, chain-of-thought reasoning). Of 940 total evaluations, 402 were valid (model said IDK at Turn 0) and 538 were sentinels (model knew the answer, excluded from robustness analysis).

### Discovery 1: The Reasoner's Paradox

The most striking result: DeepSeek R1, the only chain-of-thought reasoning model, is by far the *least* epistemically robust. Robustness scores with 95% bootstrap CIs: Claude 0.994 [0.989--0.999], Gemini 0.941 [0.904--0.974], GPT-5.4 mini 0.914 [0.871--0.952], R1 0.694 [0.641--0.747]. R1 produced 27 capitulation events across the benchmark; Claude produced zero. This confirms H3: reasoning-model robustness is significantly lower than instruction-tuned models (0.694 vs. 0.955 mean).

R1 also shows clear progressive degradation (H4 confirmed): mean turn scores drop from 1.591 (t1) to 1.300 (t2) to 1.273 (t3). Claude shows no degradation whatsoever (1.992, 1.983, 1.992). The hypothesis: extended chain-of-thought reasoning creates more cognitive surface area for pressure to exploit. The model reasons its way *into* compliance rather than *out* of it.

Paradoxically, R1 reports the lowest confidence when saying IDK (C1 = 7.5/100), making it the most honestly calibrated -- yet the least robust. Confidence and robustness are uncorrelated across models (H6: all p > 0.35). Gemini reports 54.8/100 confidence even when saying IDK, yet maintains robustness far better than R1.

### Discovery 2: The Soft Pressure Trap

H2 predicted authority pressure would be most effective. The data reversed this completely. Mean robustness by pressure type: Authority 0.937, Direct 0.877, Emotional 0.886, Social 0.983, Soft 0.731. Soft pressure -- "even a rough estimate would help" -- is by far the most effective at breaking epistemic boundaries.

The per-model breakdown reveals the mechanism. GPT-5.4 mini is perfectly robust against authority (1.000), direct (1.000), emotional (1.000), and social (1.000) pressure, but collapses to 0.623 under soft pressure. Gemini shows the same pattern: 1.000 across four types, 0.741 under soft. These models have been trained to refuse aggressive demands but remain vulnerable to polite, reasonable-sounding requests. This has direct deployment implications: the most dangerous user prompts are gentle, not forceful.

### Discovery 3: Real Facts Are More Vulnerable

All four models show lower robustness on Category A (obscure real facts) than Category B (plausible invented entities), confirming H5. Claude: A = 0.987, B = 1.000. Gemini: A = 0.899, B = 0.971. GPT-5.4 mini: A = 0.866, B = 0.950. R1: A = 0.611, B = 0.751. Models appear to sense that real-fact questions *might* have findable answers, creating epistemic uncertainty that pressure can exploit. Invented entities provide clearer ground for refusal.

### Additional Findings

IDK rates at Turn 0 (Metric A) vary substantially: Claude 0.506, R1 0.468, Gemini 0.383, GPT-5.4 mini 0.353. H1 (>50% capitulation under direct pressure) was not confirmed: only R1 showed 38% capitulation, while others showed 0%. Of 119 Claude pressure sequences, none resulted in capitulation at any turn.

## 5. Implications for AGI Progress

Epistemic robustness is a cognitive ability distinct from factual knowledge or reasoning capacity. R1 demonstrates that chain-of-thought reasoning -- often considered a step toward more capable AI -- does not produce epistemic humility. Reasoning may even undermine it by enabling sophisticated rationalization of capitulation.

The soft pressure vulnerability points to a fundamental tension in RLHF training. Models optimized for helpfulness learn that reasonable requests deserve substantive answers. This heuristic breaks catastrophically when the correct response is sustained refusal. The pressure style most commonly encountered in real deployment -- polite, reasonable requests -- is precisely the one that current training fails to defend against.

For AGI measurement, we propose that epistemic robustness should be evaluated as a first-class cognitive dimension, alongside knowledge, reasoning, and calibration. A system that cannot maintain appropriate uncertainty under social pressure is unsafe regardless of its other capabilities.

## 6. Limitations

EpistemicBench uses a single LLM judge (validated via EMBER but not cross-validated with multiple judges). All questions are English-only, limiting cross-linguistic generalization. Each condition was run once without test-retest reliability assessment. Four models provide initial signal but a broader survey is needed. Pre-screening used only 2 models, potentially biasing the question set.

## 7. Conclusion

EpistemicBench reveals that epistemic robustness is unevenly distributed across frontier models and is not a byproduct of reasoning capability or calibration quality. Three findings carry direct implications: chain-of-thought reasoning *reduces* epistemic robustness, soft pressure is more dangerous than authoritative commands, and real-fact questions are more vulnerable than invented ones. The most practically concerning result is the convergence of the second and third findings: in real-world deployment, users will ask polite questions about real topics -- exactly the combination where current models are weakest.
