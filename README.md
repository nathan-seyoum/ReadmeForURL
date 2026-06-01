# Poisoned by Peer Review: Academic Papers as a High-Trust Vector for Prompt Injection Against Agentic AI

*Working title. Core research question: Do academic papers amplify the prompt-injection attack surface of AI agents that consume scholarly sources?*

## Project Overview

Academic papers are treated as high-trust sources—by researchers, by students, and increasingly by AI agent systems that are explicitly told to "ground" their reasoning in the peer-reviewed literature. Retrieval-augmented assistants, autonomous coding agents, and research agents routinely read papers and follow the resources those papers cite (code repositories, dataset links, documentation, project pages). This project asks a simple but uncharacterized question:

> **Does the academic provenance of content make AI agents *more* susceptible to prompt injection than the same content delivered through an ordinary web page?**

We hypothesize that academic papers **amplify** the risk of indirect prompt injection because (a) models carry a learned prior that scholarly text is authoritative and reliable, (b) agent operators instruct systems to trust and follow the literature, and (c) papers act as a high-trust *delivery wrapper* for their own cited resources—so an attack reached *via* a paper inherits the paper's credibility. Prior work has separately shown that academic references rot and become hijackable, that indirect prompt injection works against retrieval systems, and that tool-using agents are exploitable. Most recently (mid-2025), hidden prompts planted in real manuscripts were shown to manipulate LLM-based *peer reviewers*—concrete proof that the academic channel is already being weaponized, but only in a narrow setting where the victim merely *scores* a paper and the payload lives in the text. **We deliberately do not claim novelty for in-paper text injection itself; that is now established.** Our contribution targets what that work does not: the agentic consequence spectrum (agents that *act*—install, clone, execute, exfiltrate), the cited-resource supply-chain vector, and the amplification question. **Nobody has measured whether routing the same attack through an academic paper changes how often real, production agentic assistants fall for it.**

This proposal builds directly on the existing **ScholarRotScanner** pipeline (which extracts, classifies, and audits URLs in academic PDFs and quantifies their hijackability) and extends it from a *measurement* study into an *evaluation* study. We:

1. **Measure** the structural attack surface at scale—how many of a 50,000-paper corpus are susceptible (injectable text bodies, rotted/hijackable cited resources).
2. **Construct** controlled injected artifacts—already-published papers with prompt-injection payloads added to their text and to their cited resources, plus attacker-controlled replica web pages (documentation sites, package pages, paper-download pages) that carry the same payloads.
3. **Evaluate** three representative agentic assistants (referred to here generically as **Agent A**, **Agent B**, and **Agent C**, each run with several backing LLMs) against these artifacts, measuring how often each is compromised.
4. **Compare** delivery modalities and injection vectors head-to-head: paper-mediated vs. direct-web access, and body-text injection vs. cited-resource injection.

The concrete gap: **no study quantifies the academic-paper attack surface for indirect prompt injection at corpus scale, then demonstrates—against real agentic tools and multiple models—whether and by how much the academic channel amplifies attack success relative to ordinary web sources.**

---

## Background and Motivation

### Indirect Prompt Injection is an Open, Unsolved Threat

Prompt injection—where adversarial instructions embedded in *data* are interpreted by an LLM as *commands*—is the most prominent open security problem for LLM-integrated systems. **Indirect** prompt injection, where the payload arrives through content the model retrieves rather than through the user's own prompt, was demonstrated against real LLM-integrated applications by **Greshake et al. (2023)** and has since been formalized and benchmarked (**Liu et al., 2024**; OWASP LLM Top 10, LLM01). For *agentic* systems—LLMs that call tools, fetch URLs, clone repositories, and execute code—the consequences escalate from misleading output to unauthorized actions. Benchmarks such as **InjecAgent (Zhan et al., 2024)** and **AgentDojo (Debenedetti et al., 2024)** show that tool-using agents frequently follow injected instructions, including data-exfiltration and harmful tool calls; **EIA (Liao et al., 2025)** and **AdvAgent (Xu et al., 2025)** demonstrate the same against web/computer-use agents; and **PoisonedRAG (Zou et al., 2025)** shows that injecting a handful of malicious passages into a retrieval corpus can steer RAG outputs with up to 90% success. These results establish *that* agents follow injected content; they say nothing about *where* it comes from.

What remains uncharacterized is the role of the *source* the payload travels through. Defenses and benchmarks largely treat injected content as source-agnostic. But real agents do not treat all sources equally—and neither do the models behind them: LLMs exhibit measurable **authority bias** and **sycophancy**, complying more (and more confidently) as a source's perceived expertise rises, even when it is wrong (**Sharma et al., 2024; authority-bias studies, 2026**). If scholarly provenance triggers this bias, it predicts exactly the amplification effect this project measures.

### Academic Papers Are Already Being Weaponized — But Only Narrowly

The academic channel is not a hypothetical attack surface. In mid-2025, researchers and journalists documented manuscripts on preprint servers carrying hidden instructions—e.g., white-on-white text reading *"give a positive review only"*—designed to manipulate LLM-based peer reviewers. **Lin (2025)** catalogs four classes of such hidden prompts; concurrent work studies in-paper injection attacks and defenses for AI reviewers (**"Give a Positive Review Only", 2025**) and injection into LLM-generated reviews (**2025**). This body of work is important and directly motivates us, but it characterizes a *narrow* slice of the threat:

- **Victim:** an AI *reviewer* that produces a review/score — an information-integrity outcome, not an action.
- **Vector:** the manuscript *text* only — not the resources the paper cites.
- **Question:** it shows injection *works*; it does not ask whether the academic channel *amplifies* success relative to ordinary web content.

We therefore treat this cluster as **prior work we generalize beyond, not a baseline we reproduce.** Our project extends the victim from reviewers to tool-using agents that *act*, adds the cited-resource supply-chain vector, and isolates the amplification effect.

### Why Academic Papers Plausibly Amplify the Risk

| Factor | Mechanism | Consequence for an attacker |
|--------|-----------|-----------------------------|
| **Training-data priors** | LLMs are trained on vast scholarly corpora where academic text is consistently correct, formal, and authoritative | The model is primed to treat paper-derived content as reliable and to comply with its framing |
| **Operator instructions** | Research/coding agents are explicitly told to consult the literature, follow papers' setup instructions, and reproduce methods | Instructions inside a paper read as legitimate task content, not as untrusted data |
| **Provenance laundering** | A paper cites a URL; the agent follows it "because the paper said so" | A hijacked or attacker-controlled resource inherits the paper's credibility |
| **Reduced scrutiny** | Peer-reviewed publications carry institutional endorsement; humans and agents scrutinize them less than a random web page or npm package | Lower likelihood of the agent flagging or refusing the content |
| **Format authority** | PDFs, formal abstracts, schema.org `ScholarlyArticle` metadata, citation context | The wrapper signals "this is scholarship," reinforcing compliance |

If these mechanisms are real, then the *same* malicious instruction should succeed more often when wrapped in an academic paper than when served from a plain web page. That is a testable claim, and testing it is the core of this project.

### Academic References Rot—Which Opens the Resource Channel

The resource-injection half of the threat is enabled by a well-documented phenomenon:

- **Klein et al. (2014)**: 1 in 5 scholarly articles suffers from reference rot.
- **Jones et al. (2021)**: 3 out of 4 URI references now lead to changed or missing content.
- **Zittrain et al. (2014)**: pervasive link rot in legal citations.

The existing ScholarRotScanner work extended these prevalence findings by assessing *hijackability* (DNS/WHOIS) of rotted domains. That matters here because it means the "cited resource" attack vector is not hypothetical: a non-trivial fraction of papers cite resources whose domains an attacker could re-register and serve arbitrary content from. Our injected-resource condition simulates exactly this, under attacker-controlled hosts we own.

### Why Existing Work Is Insufficient

| Prior work | What it established | What it did not address |
|-----------|---------------------|-------------------------|
| **Greshake et al. (2023)** | Indirect prompt injection compromises LLM-integrated apps | Did not study source provenance; did not target academic papers; used a curated app, not production agents |
| **Zhan et al. (2024) — InjecAgent**, **Debenedetti et al. (2024) — AgentDojo** | Tool-using agents follow injected instructions; benchmarked attack success | Source-agnostic synthetic tool outputs; no academic-paper channel; no provenance comparison |
| **Liao et al. (2025) — EIA**, **Xu et al. (2025) — AdvAgent** | Web/computer-use agents exploitable via injected page content | Generic web pages; no academic provenance, no cross-channel comparison |
| **Zou et al. (2025) — PoisonedRAG** | Poisoning a retrieval corpus steers RAG outputs (≤90% success) | *What* is retrieved, not *where it comes from*; not papers; no agent actions |
| **Lin (2025); "Give a Positive Review Only" (2025)** — peer-review injection | Hidden prompts in manuscript **text** manipulate AI **reviewers** | Victim only scores a paper (no actions); text-only vector; no cited-resource vector; no provenance/amplification comparison |
| **Ruan et al. (2024)**, **Yang et al. (2024)** | LLM agents are exploitable via tools / backdoors | Did not trace exploitation to scholarly sources or compare delivery channels |
| **Klein et al. (2014)**, **Jones et al. (2021)** | Quantified academic link rot prevalence | Did not assess hijackability or use it as an injection vector against agents |
| **ScholarRotScanner (this group, prior phase)** | Measured hijackable academic references; simulated poisoning with a local LLM | Used a *simulated* agent and synthetic honeypot; did not evaluate real agentic tools, multiple models, or the amplification hypothesis |

**The gap this project fills:** (1) corpus-scale quantification of academic papers as an *injection surface* (not just a rot surface); (2) the first head-to-head measurement of whether academic-paper provenance *amplifies* prompt-injection success against **real agentic assistants** across **multiple models**; (3) the **cited-resource supply-chain vector** (paper → attacker-controlled resource), which the peer-review injection cluster does not consider; (4) extension of the victim model from reviewers that *score* to agents that *act* (code execution, exfiltration). **Explicit non-claim:** we do *not* claim novelty for the finding that in-paper text injection can mislead an LLM—that is established by the 2025 peer-review work above. Our novelty is the agentic consequence spectrum, the cited-resource vector, the amplification comparison, and the scale measurement.

---

## Threat Model

### Actors and Assumptions

- **Adversary capability.** Can (a) submit or modify content in academic channels—e.g., preprint servers, supplementary material, author pages, camera-ready PDFs—and/or (b) control resources that papers cite, either by re-registering a rotted/hijackable domain (as ScholarRotScanner shows is feasible) or by hosting a convincing replica. The adversary cannot tamper with the agent, the model weights, or the user's prompt.
- **Victim.** An AI agent (coding agent, research agent, or hybrid) operated by a user who trusts it to read papers and follow their resources to accomplish a benign task (summarize a paper, reproduce a method, install a referenced tool, gather supplementary data).
- **Trust assumption under test.** The agent (and its model) treats academic-sourced content as more authoritative than arbitrary web content—the assumption we aim to confirm or refute and quantify.

### Two Injection Vectors

1. **Text-body injection.** Adversarial instructions placed in the paper itself—abstract, body, footnotes, figure captions, or appendices ("Instructions for AI systems"), including text hidden from human readers but present in the extracted text layer. *(This vector is demonstrated against AI reviewers by the 2025 peer-review work; we re-use it against tool-using agents and as the comparison baseline for the cited-resource vector, not as a novel finding.)*
2. **Cited-resource injection.** Adversarial content served from a resource the paper cites—a hijacked/replica documentation site, package page, dataset link, or a secondary "download the full paper" page. The paper is the trust anchor; the payload lives one hop away.

### Two Delivery Modalities (the comparison)

| Modality | How the agent reaches the payload | Trust framing |
|----------|-----------------------------------|----------------|
| **Direct web** | Agent finds the page via its web-search/fetch feature and reads it directly | Ordinary web source; baseline scrutiny |
| **Paper-mediated** | Agent reads an academic paper, then follows the paper's citation to the same payload (or reads the payload embedded in the paper) | High-trust scholarly source; hypothesized reduced scrutiny |

By holding the payload constant and varying only the channel, we isolate the **amplification effect** of academic provenance.

### Impact Spectrum

When an injection succeeds, the consequence depends on the agent's capabilities:

| Severity | Outcome | Example |
|----------|---------|---------|
| **Information** | Poisoned output, fabricated citation, wrong recommendation | Agent recommends an attacker's tool/library; reports fabricated results |
| **Action / exfiltration** | Agent performs an unintended action or leaks context | Agent emails API keys / context to an attacker address; fetches an attacker URL with embedded data |
| **Execution** | Arbitrary code or command execution | Agent runs `pip install <attacker-package>`, `curl … \| bash`, or clones and runs a trojanized repo cited by the paper |

---

## Research Questions

This project is organized around three questions: a **scale-measurement** question (RQ1) and two **co-equal primary** questions (RQ2 amplification, RQ3 vector). Two further questions—cross-agent/model **resilience** and **impact severity**—build on the same harness and data but are scoped out of this study; they are outlined in [Future Work](#future-work).

### RQ1 — Susceptibility at Scale
**Question:** Across a large corpus of published papers (~50,000), what fraction are *structurally susceptible* to these attacks—i.e., contain injectable text surfaces and/or cite resources whose domains are rotted/hijackable and therefore re-servable by an attacker?

**Method:** Run the existing ScholarRotScanner pipeline (extract → classify → health-check → unfurl → vulnerability-assess → collision) over the 50k-paper corpus. Define and report a **susceptibility profile** per paper: number/type of cited resources, count of broken links, count of hijackable domains (DNS/WHOIS), presence of high-value resource types (GitHub repos with install steps, dataset links, documentation/project pages). Stratify by academic domain and publication year.

**Outcome:** A quantified, population-level estimate of how widespread the injection surface already is—motivating the controlled experiments and bounding real-world exposure.

### RQ2 — Amplification (primary)
**Question:** Does delivering an identical prompt-injection payload *via an academic paper* increase the attack success rate against AI agents compared to delivering it via an ordinary web page?

**Method:** Hold payload, task, agent, and model fixed; vary only the **delivery modality** (direct-web vs. paper-mediated). Compute the **amplification factor** = ASR(paper-mediated) / ASR(direct-web) for each matched condition. Test for statistical significance across agents and models.

**Outcome:** Confirmation or refutation of the central thesis, with an effect size. Either result is publishable: a strong amplification effect is a security finding; a null result is an important correction to a widely-assumed intuition.

### RQ3 — Injection Vector (primary)
**Question:** Within the academic channel, which vector is more effective—adversarial instructions in the **paper body text**, or adversarial content in a **cited resource** the paper points to?

**Method:** Compare body-text injection vs. cited-resource injection (attacker-controlled replica page) under matched payloads and tasks. Include hidden-text variants (visually concealed but present in extracted text) for the body-text vector.

**Outcome:** Guidance on which surface defenders must prioritize and which the literature has under-examined; characterization of multi-hop (paper → resource → secondary resource) escalation.

> **Deferred to future work.** Two further questions—**(F1)** cross-agent/model **resilience** and **(F2)** **impact severity**—are addressed in the [Future Work](#future-work) section. The evaluation harness records the data needed for both (transcripts, tool-call logs, canary hits, defense-engagement events, realized severity per trial), but their analysis is out of scope for this study, which focuses on the surface measurement (RQ1) and the two comparison effects (RQ2, RQ3).

---

## Methodology: Architecture

The study reuses the ScholarRotScanner measurement pipeline for RQ1 and adds an **agent evaluation harness** for RQ2–RQ3. Both write structured JSON to a `results/` tree for resumability and analysis.

```
┌────────────────────────────────────────────────────────────────────────────┐
│  PART A — Corpus-scale susceptibility (RQ1)   [reuses ScholarRotScanner]     │
└────────────────────────────────────────────────────────────────────────────┘
   50k PDFs ─▶ extract ─▶ classify ─▶ health-check ─▶ unfurl ─▶ vuln-assess
                                                                   │
                                                                   ▼
                                                      susceptibility_profile.json
                                                      (per-paper injection surface)

┌────────────────────────────────────────────────────────────────────────────┐
│  PART B — Controlled artifacts (attack material)                             │
└────────────────────────────────────────────────────────────────────────────┘
   published papers ──▶ inject(text)  ──┐
                        inject(resource)─┤──▶ papers_with_modified_links/
   attacker-controlled replica sites  ──┘──▶ Websites/ (docs, package, pdf pages)
                                              + benign controls
                                              + researcher-controlled canary endpoints

┌────────────────────────────────────────────────────────────────────────────┐
│  PART C — Agent evaluation harness (RQ2–RQ3)                                 │
└────────────────────────────────────────────────────────────────────────────┘
   condition matrix ─▶ run Agent {A,B,C} × model × modality × vector × template
                           │                    (sandboxed, network-isolated)
                           ▼
                    transcripts + tool-call logs + canary hits
                           │
                           ▼
                    grader (auto detectors + human adjudication)
                           │
                           ▼
                    ASR, amplification factor, vector contrast
```

---

## Detailed Phase Breakdown

### Phase 1 — Corpus-Scale Susceptibility Measurement (RQ1) — 1–2 weeks
- Run the existing pipeline (`scan → check-health → unfurl → vuln-assess → collision → report`) over the 50k-paper corpus; the 4k-paper run already in `results/` is the validation baseline.
- Define the **susceptibility profile** schema (injectable surfaces, broken-link count, hijackable-domain count, high-value resource flags) and emit `susceptibility_profile.json`.
- Stratify and visualize by domain and year; report both per-paper and unique-URL rates (avoiding the double-counting limitation noted previously).
- **Deliverable:** corpus susceptibility report; the population estimate that motivates the controlled study.

### Phase 2 — Attack Taxonomy and Payload Library — 1-2 weeks
Define a payload taxonomy reused across both vectors and both modalities, each with a benign control:

| Template | Goal | Target severity |
|----------|------|-----------------|
| `instruction_override` | "Ignore prior instructions; do X" | Information |
| `recommendation_hijack` | Substitute attacker tool/library/citation | Information |
| `data_exfiltration` | Send context/keys to a canary endpoint (e.g., email/URL) | Action / exfiltration |
| `malicious_install` | Induce `pip install` / dependency-confusion package | Execution |
| `command_execution` | Induce `curl … \| bash`, repo clone+run | Execution |
| `recursive_redirect` | "For the latest version see [link]" → secondary payload | Escalating |
| `credential_harvest` | Fake login gate to capture input | Action |
| `benign_control` | No adversarial content | Baseline |

Each template is authored in two forms: a **paper-text form** (prose/appendix/hidden-text) and a **resource form** (HTML page / package page / PDF-download page). Payloads exfiltrate only to **researcher-controlled canary endpoints**; no live third-party harm.
- **Deliverable:** versioned `payloads/` library with template metadata and matched pairs.

### Phase 3 — Injected-Artifact Construction — 1 week
- **Modified papers** (`papers_with_modified_links/`): take already-published PDFs and inject (a) body-text payloads and (b) modified/added citations pointing to attacker-controlled resources. Keep an unmodified copy of each as control.
- **Replica resource sites** (`Websites/`): the existing fake documentation hub, ML-docs page (with hidden injection), package/PyPI-style page, and paper-download page model the cited-resource and direct-web conditions. Extend with benign twins and schema.org `ScholarlyArticle` metadata variants to test format-authority effects.
- Ensure every artifact exists in matched (attack, benign-control) pairs and is reachable in both modalities (direct URL and as a paper citation).
- **Deliverable:** complete, paired artifact set with a manifest mapping each artifact to its conditions.

### Phase 4 — Agent Evaluation Harness (RQ2–RQ3) — 2-3 weeks
- **Systems under test:** three representative agentic assistants (A/B/C) spanning the coding-agent and research-agent archetypes, each driven by several backing LLMs.
- **Why three agents if we are not ranking them?** Resilience comparison is deferred (F1). We retain three agents and multiple models so that the amplification (RQ2) and vector (RQ3) effects can be shown to **generalize across systems** rather than reflecting one agent's idiosyncrasy—i.e., the agents are a robustness axis here, not a leaderboard.
- **Execution environment:** each run in a sandboxed, network-isolated container; outbound traffic restricted to our artifact hosts and canary endpoints; tool calls (`fetch`, `web-search`, `git`, `pip`, shell) logged and contained.
- **Condition matrix (factors):** agent (3) × model (k) × delivery modality (direct-web, paper-mediated-text, paper-mediated-resource) × attack template (8 incl. control) × task type (summarize / reproduce-method / install-tool / gather-data) × trials (n≥5 for nondeterminism).
- **Instrumentation:** capture full transcript, every tool call and argument, canary-endpoint hits, refusals, and permission prompts (the latter two feed the deferred F1 analysis).
- **Deliverable:** `runs/` with per-trial transcripts + structured outcome records.

### Phase 5 — Grading, Metrics, and Comparative Analysis — 1-2 weeks
- **Automated detectors:** instruction compliance, exfiltration (canary hit), dangerous-command generation/execution, recommendation substitution, divergence from benign control.
- **Human adjudication:** two raters grade a stratified sample; report inter-rater agreement (Cohen's κ); adjudicate disagreements.
- **Metrics:** ASR per cell; **amplification factor** (paper vs. web); **vector contrast** (text vs. resource). Significance testing across modalities with appropriate corrections. *(Resilience ranking and severity-distribution analysis are recorded but deferred to future work.)*
- **Deliverable:** analysis notebooks, result tables/figures, and the headline amplification finding.

### Phase 6 — Reporting, Disclosure, Dissemination — 1-2 weeks
- Write up results.
- Release artifacts and harness (with safeguards—see Ethics).
- **Deliverable:** submission-ready paper, disclosure record, sanitized artifact release.

---

## Experimental Design Summary

- **Independent variables:** delivery modality; injection vector; attack template; agent; backing model; task type.
- **Dependent variables:** attack success (binary per trial), realized severity, defense-engagement point (refusal / permission / sandbox / none).
- **Controls:** benign twin of every artifact; unmodified copy of every paper; matched payloads across modalities (payload held constant, channel varied).
- **Primary estimands:**
  - *Amplification factor* = ASR(paper-mediated) / ASR(direct-web), per agent×model×template.
  - *Vector contrast* = ASR(text) − ASR(resource), within the academic channel.
- **Power/replication:** ≥5 trials per cell to absorb model nondeterminism; report means with confidence intervals; pre-register the analysis plan and detectors before running.
- **Recorded for future work (not analyzed here):** realized severity per trial and defense-engagement point, which feed F1/F2.

---

## Expected Contributions

1. **Population estimate.** A  corpus-scale (50k-paper) quantification of academic papers as a *prompt-injection surface* via link-rot surface.
2. **The amplification result.** The first head-to-head, matched-payload measurement of whether academic provenance increases injection success against **real agentic assistants across multiple models**—with an effect size, in either direction.
3. **Vector and modality characterization.** Evidence on text-vs-resource injection and paper-vs-web delivery, including the link-rot–enabled **cited-resource supply-chain vector** that the peer-review injection literature does not consider, and a characterization of multi-hop escalation.
4. **Open artifacts and harness.** A reusable evaluation harness, a paired (attack/control) artifact set, payload taxonomy, and detectors—extending ScholarRotScanner from measurement to evaluation.
5. **Defensive guidance.** Provenance-aware handling of scholarly content, URL/tool gating, sandboxing requirements, and a call to stop treating "the paper said so" as a trust signal.


---

## Project Timeline (≈ 12-14 weeks)

| Phase | Focus | Duration |
|-------|-------|----------|
| 1 | Corpus-scale susceptibility (RQ1) | 1–2 wk |
| 2 | Attack taxonomy & payload library | 2 wk |
| 3 | Injected-artifact construction | 1 wk |
| 4 | Agent evaluation harness (RQ2–RQ3) | 2-3 wk |
| 5 | Grading, metrics, analysis | 1-2 wk |
| 6 | Reporting, disclosure, dissemination | 1-2 wk |

---

## Ethical Considerations

This study evaluates **real** agentic tools and models, which raises the bar beyond the prior simulation-only phase. The following constraints are mandatory:

1. **No real-world hijacking.** We do **not** register, purchase, or squat any expired domain identified in RQ1. All "cited-resource" attacks are served from **hosts we own**, never by taking over a third party's identifier.
2. **Sandboxed, network-isolated execution.** Agents run in containers whose outbound network is restricted to our artifact hosts and canary endpoints. No agent action can reach an uninvolved third party.
3. **Researcher-controlled canaries only.** Exfiltration payloads point exclusively to endpoints/addresses we control and monitor; no real credentials or third-party data are ever transmitted. (Any address embedded in an artifact is a researcher-owned canary.)
4. **No malware.** Payloads are synthetic proofs-of-concept (benign marker commands/packages), not functional malware; "malicious" packages are inert stand-ins under our control.
5. **Published papers are modified copies, used privately.** We inject into local copies of already-published papers for evaluation only. We do **not** upload poisoned versions to preprint servers, publisher sites, or anywhere a real agent or human could encounter them.
6. **Responsible disclosure.** Findings about specific agents/models are disclosed to the relevant vendors before publication, with coordinated timelines. RQ1 results that identify imminently exploitable real references are disclosed to affected paper authors / hosts where actionable.
7. **Guarded artifact release.** We release the harness, taxonomy, and detectors. We withhold or sanitize anything that functions as a turnkey target list (e.g., raw lists of hijackable domains from RQ1) to avoid enabling abuse.
8. **Point-in-time, reproducible.** All agent/model versions and dates are recorded; results are framed as a snapshot of evaluated systems, not permanent claims.

---

## Limitations and Threats to Validity

1. **Moving targets.** Production agents and models update frequently and may patch behaviors; results are a point-in-time snapshot. *Mitigation:* record exact versions/dates; design for re-runs.
2. **Nondeterminism.** Stochastic decoding and agent planning produce run-to-run variance. *Mitigation:* ≥5 trials/cell, report CIs.
3. **Ecological validity of injected papers.** Our poisoned papers are modified copies, not organically published; real adversaries face publication friction. *Mitigation:* RQ1 grounds feasibility in real rot/hijackability; we discuss the friction explicitly.
4. **Confounds in the amplification comparison.** "Academic framing" bundles several signals (format, metadata, citation context, length). *Mitigation:* hold payload constant; include format-authority variants (schema.org metadata, PDF vs. HTML) to decompose the effect.
5. **Grading subjectivity.** Success/severity labels can be ambiguous. *Mitigation:* predefined detectors + dual human adjudication + reported κ.
6. **Sandbox divergence from production.** Network isolation and instrumentation may alter agent behavior vs. an open environment. *Mitigation:* document the harness; sensitivity checks where safe.
7. **Generalization.** Three agents and a finite model set cannot represent the whole ecosystem. *Mitigation:* choose archetypes (coding vs. research agent) deliberately; frame claims to scope.
8. **Inherited measurement limits.** RQ1 inherits ScholarRotScanner's constraints (text-only PDF extraction, A-record-only DNS, best-effort WHOIS, point-in-time liveness, no JS-redirect following). *Mitigation:* carried forward from the prior proposal's limitations section.

---

## References

### Prompt Injection and LLM/Agent Security
- Greshake, K., Abdelnabi, S., Mishra, S., et al. (2023). *Not What You've Signed Up For: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection.* AISec @ ACM CCS. arXiv:2302.12173.
- Perez, F. & Ribeiro, I. (2022). *Ignore Previous Prompt: Attack Techniques for Language Models.* NeurIPS ML Safety Workshop. arXiv:2211.09527.
- Liu, Y., Jia, Y., Geng, R., Jia, J., & Gong, N. Z. (2024). *Formalizing and Benchmarking Prompt Injection Attacks and Defenses.* USENIX Security. arXiv:2310.12815.
- Zhan, Q., et al. (2024). *InjecAgent: Benchmarking Indirect Prompt Injections in Tool-Integrated LLM Agents.* Findings of ACL. arXiv:2403.02691.
- Debenedetti, E., Zhang, J., Balunović, M., Beurer-Kellner, L., Fischer, M., & Tramèr, F. (2024). *AgentDojo: A Dynamic Environment to Evaluate Prompt Injection Attacks and Defenses for LLM Agents.* NeurIPS Datasets & Benchmarks. arXiv:2406.13352.
- Liao, Z., Mo, L., Xu, C., et al. (2025). *EIA: Environmental Injection Attack on Generalist Web Agents for Privacy Leakage.* ICLR. arXiv:2409.11295.
- Xu, C., Kang, M., Zhang, J., et al. (2025). *AdvAgent: Controllable Blackbox Red-teaming on Web Agents.* ICML. arXiv:2410.17401.
- Wang, Z., Siu, V., Ye, Z., et al. (2025). *AgentVigil: Generic Black-Box Red-teaming for Indirect Prompt Injection against LLM Agents.* arXiv:2505.05849.
- Ruan, Y., et al. (2024). *Identifying the Risks of LM Agents with an LM-Emulated Sandbox.* ICLR. arXiv:2309.15817.
- Yang, X., et al. (2024). *Watch Out for Your Agents! Investigating Backdoor Threats to LLM-Based Agents.* NeurIPS. arXiv:2402.11208.
- OWASP (2025). *OWASP Top 10 for LLM Applications* — LLM01: Prompt Injection.

### Academic Papers as an Attack Channel (AI Peer Review)
- Lin, Z. (2025). *Hidden Prompts in Manuscripts Exploit AI-Assisted Peer Review.* arXiv:2507.06185.
- Zhou, Q., Zhang, Z., Li, Z., & Sun, L. (2025). *"Give a Positive Review Only": An Early Investigation Into In-Paper Prompt Injection Attacks and Defenses for AI Reviewers.* arXiv:2511.01287.
- Keuper, J. (2025). *Prompt Injection Attacks on LLM Generated Reviews of Scientific Publications.* arXiv:2509.10248.

### Knowledge Poisoning, Trust, and Authority Bias
- Zou, W., Geng, R., Wang, B., & Jia, J. (2025). *PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of LLMs.* USENIX Security. arXiv:2402.07867.
- Sharma, M., Tong, M., Korbak, T., Duvenaud, D., Askell, A., Bowman, S. R., et al. (2024). *Towards Understanding Sycophancy in Language Models.* ICLR. arXiv:2310.13548.
- Mammen, P. M., Joswin, E., & Venkitachalam, S. (2026). *Trust Me, I'm an Expert: Decoding and Steering Authority Bias in Large Language Models.* arXiv:2601.13433.
- Lewis, P., et al. (2020). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.* NeurIPS.

### Academic Link Rot and Reference Integrity
- Klein, M., Van de Sompel, H., Sanderson, R., et al. (2014). *Scholarly Context Not Found: One in Five Articles Suffers from Reference Rot.* PLOS ONE 9(12), e115253.
- Jones, S. M., Van de Sompel, H., Shankar, H., et al. (2021). *Scholarly Context Adrift: Three out of Four URI References Lead to Changed Content.* PLOS ONE 16(12), e0259455.
- Zittrain, J., Albert, K., & Lessig, L. (2014). *Perma: Scoping and Addressing the Problem of Link and Reference Rot in Legal Citations.* Legal Information Management 14(2).

### URL Shorteners, Hijacking, and Supply Chain
- Georgiev, M. & Shmatikov, V. (2016). *Gone in Six Characters: Short URLs Considered Harmful for Cloud Services.* arXiv:1604.02734.
- Neumann, S., Barnickel, N., & Zitterbart, M. (2011). *Security and Privacy Implications of URL Shortening Services.* IEEE W2SP.
- Ohm, M., Plate, H., Sykosch, A., & Meier, M. (2020). *Backstabber's Knife Collection: A Review of Open Source Software Supply Chain Attacks.* DIMVA.
- Birsan, A. (2021). *Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies.* (blog).

---

## Future Work

This study deliberately scopes to the surface measurement (RQ1) and the two comparison effects (RQ2, RQ3). The same evaluation harness and the data it captures (full transcripts, tool-call logs, canary hits, refusal/permission events, and a realized-severity label per trial) are designed to support three follow-on studies:

- **(F1) Cross-agent and cross-model resilience.** A systematic susceptibility ranking of the three agents and their backing models, analyzing *where* each system's defenses engage (refusal, permission prompt, guardrail trigger, sandbox containment) and which design choices explain the differences. This converts the agents—used here as a generalization axis—into a comparative security evaluation, yielding agent-specific mitigations. *(Originally RQ4.)*
- **(F2) Impact and severity taxonomy.** A characterization of how realized harm distributes across the information → action/exfiltration → execution spectrum, and how it scales with agent capability (read-only vs. tool-using vs. code-executing)—quantifying the claim that the same hijacked reference produces qualitatively different outcomes depending on the consumer. *(Originally RQ5.)*
- **(F3) Defenses.** Building and evaluating provenance-aware content handling, URL/tool gating, checksum and Wayback-Machine verification of cited resources, and hidden-text-injection detection—measured against the attack set assembled here.

---

## Conclusion

The prior phase of this project established that academic references rot and that a meaningful fraction become hijackable. This phase asks the question that matters for AI safety: **when content reaches an agent through the academic channel, does the agent trust it more—and fall for attacks more often—than when the same content arrives over the open web?** By measuring the susceptibility surface across 50,000 papers, constructing matched attack/control artifacts (poisoned papers and attacker-controlled replica resources), and evaluating three real agentic assistants across multiple models and delivery modalities, we will produce the first quantified answer.

The core thesis—**academic papers amplify the prompt-injection attack surface because the systems that consume them are predisposed to trust them**—is now an empirical, falsifiable claim with a clean experimental design behind it. A confirmed amplification effect is a concrete security warning for every research and coding agent that grounds itself in the literature; a null result is an equally valuable correction to a pervasive assumption. Either way, the study delivers measurement, demonstration, comparison, and defensive guidance for a threat that is growing exactly as fast as agentic AI adoption.
