# 🗄️ AI Models & Data Security

> **A deep dive into the security risks hidden inside AI training pipelines, model-building decisions, and the opaque nature of trained model weights — risks that begin long before a model is ever deployed.**

---

## 📚 Table of Contents

- [The Data Supply Chain Problem](#the-data-supply-chain-problem)
  - [Where Does Training Data Come From?](#where-does-training-data-come-from)
  - [The Problem of Provenance](#the-problem-of-provenance)
  - [PII in the Pipeline](#pii-in-the-pipeline)
- [Model Building: Security Implications](#model-building-security-implications)
  - [Epochs & Overfitting](#epochs--overfitting)
  - [Model Validation](#model-validation)
  - [Pruning & Quantisation](#pruning--quantisation)
  - [Federated Learning](#federated-learning)
- [Pre-Trained Models & Fine-Tuning](#pre-trained-models--fine-tuning)
  - [The Inheritance Problem](#the-inheritance-problem)
- [Model Opacity: The Black Box](#model-opacity-the-black-box)
  - [Model Cards](#model-cards)
  - [The Gaps in Documentation](#the-gaps-in-documentation)
- [Key Takeaways](#key-takeaways)
- [Quick Reference: Key Terms](#quick-reference-key-terms)

---

## 🔍 The Data Supply Chain Problem

> *"Every AI model is, at its core, a product of its training data."*

The security risks of AI don't begin at deployment — they begin in the **data pipeline**, long before a model is trained. Decisions about what data to collect, where to get it, and how to process it shape everything the model will ever do. Most of these decisions are **invisible, poorly documented, and almost entirely unaudited**.

---

### Where Does Training Data Come From?

Training a large language model requires staggering amounts of text. GPT-3 was trained on **~570GB of filtered text** — considered modest by modern standards. Data typically comes from four sources:

| Source | What It Is | Trust Profile |
|---|---|---|
| **Web Scraping** | Automated crawls of public internet content | 🔴 **Low** — no curator, content changes after collection |
| **Licensed Datasets** | Data purchased or agreed with platforms (e.g., Reddit) | 🟡 **Medium** — original users rarely consented to AI training use |
| **Synthetic Data** | AI-generated content used to train further AI | 🟡 **Variable** — ~12% of fine-tuning datasets now contain LLM-generated content |
| **Internal Corpora** | Company knowledge bases, support transcripts, clinical notes | 🟢 **Higher** — direct control, but also direct liability if mishandled |

**Common Crawl** — a free, publicly available web archive — underpins essentially every major model family:

```
GPT-3       → 60% of tokens from filtered Common Crawl
DeepSeek-V2 → Pretrained on Common Crawl
DeepSeek-V3 → 14.8 trillion tokens, Common Crawl as core source
LLaMA 4     → 40 trillion tokens across 200 languages, similar pipeline
```

> ⚠️ The word **"filtered"** is key — *how* filtering was done, by whom, and *what slipped through* is where the security story begins.

---

### The Problem of Provenance

**Data provenance** is the ability to answer three questions about any training data:

```
1. Where did it come from?
2. When was it collected?
3. Has it been modified since?
```

For most AI systems today, the honest answer to all three is: **we don't fully know.**

Major models are trained on **datasets of datasets** — huge composites assembled from hundreds of upstream sources where original attribution is lost or never recorded.

**The Data Provenance Initiative** audited over 1,800 datasets and found:
- **70%+ of licenses** on popular hosting platforms were listed as "Unspecified."
- Of those that were labelled, **66% were miscategorised** — usually listed as more permissive than they actually were.

**The ML-BOM (Machine Learning Bill of Materials)**

The software security world solved a similar problem with **SBOMs (Software Bills of Materials)** after incidents like SolarWinds. The AI equivalent is the **ML-BOM**:

| ML-BOM Contains | Purpose |
|---|---|
| Dataset sources | Know what went in |
| Licenses | Understand legal exposure |
| PII categories | Identify privacy risk |
| Filtering decisions | Understand what may have slipped through |

> 📌 Most organisations deploying third-party models today have **nothing close to an ML-BOM**.

---

### PII in the Pipeline

Large-scale, unaudited web scraping means **Personally Identifiable Information (PII) gets baked directly into model weights**. Once embedded, it is very difficult to remove.

What ends up in models:
- Medical records and health conditions
- Personal email threads
- Forum posts about political views
- Login credentials and API keys

**Real-World Evidence:**

Truffle Security scanned the **December 2024 Common Crawl archive** (400TB, 2.67 billion web pages) and found:

```
~12,000 live, verified API keys and passwords
```

> ⚠️ With the right prompt, a model trained on this data can sometimes be coaxed into surfacing training content — including live credentials. **This is not a bug introduced by an attacker. It's a consequence of what went into training, and no patch fixes it once the model is deployed.**

**Legal Tension:**

| Regulation | Requirement | AI Reality |
|---|---|---|
| EU GDPR | Data minimisation — collect only what's necessary | "More data is always better" drives pre-training |

---

## 🏗️ Model Building: Security Implications

### Epochs & Overfitting

An **epoch** is one complete pass of the training algorithm through the entire dataset. Models are trained across many epochs, with parameters adjusting each pass.

```
More epochs → Better generalisation... up to a point
Too many epochs → Overfitting
```

**Overfitting** occurs when a model stops learning general patterns and starts **memorising training data**:

| Problem | Security Implication |
|---|---|
| Model memorises training examples | Increases likelihood of reproducing sensitive details when prompted |
| Poor generalisation | Fails on real-world, unseen data |

---

### Model Validation

A **validation set** (data held back from training) is used at regular intervals to test whether the model is genuinely learning or just memorising.

```
Training Accuracy ↑ + Validation Accuracy → Plateau/Drop = Overfitting Detected
```

> 🔐 From a security perspective, **validation is the quality gate** of the ML lifecycle. A model skipping thorough validation has unknown real-world behaviour — and unknown behaviour is a security risk.

---

### Pruning & Quantisation

Once trained, models often undergo **post-training compression** before deployment:

| Technique | What It Does | Security Risk |
|---|---|---|
| **Pruning** | Removes low-contribution parameters to shrink model size | Changes model behaviour post-training; rarely documented |
| **Quantisation** | Reduces weight precision (e.g., 32-bit → 8-bit) to cut memory/compute | Can silently degrade safety-aligned behaviour; backdoor defences may fail on compressed models |

> ⚠️ Research shows quantisation can **silently degrade safety mechanisms** built into a model. Defences that worked on the full-precision version may fail to detect backdoors once compressed.

These steps are often applied by **a different third-party team** packaging the model — meaning organisations inherit **unknown behaviour modifications** alongside efficiency gains.

---

### Federated Learning

**Federated Learning** trains a model across **many decentralised devices or organisations**, with each participant training locally and only sending **weight updates** (not raw data) to a central server.

```
Device A (local data) ──┐
Device B (local data) ──┼──► Central Server (aggregates weight updates) → Global Model
Device C (local data) ──┘
```

| Aspect | Detail |
|---|---|
| **Privacy Benefit** | Raw data never leaves the participant — ideal for sensitive domains like healthcare |
| **Security Trade-off** | Participants can submit **poisoned local updates** (manipulated gradients) that are very difficult to detect at the aggregation stage |

> Federated learning solves one trust problem by distributing control, but creates a different one — **who controls the aggregation, and can any participant corrupt it?**

---

## 🔧 Pre-Trained Models & Fine-Tuning

Training an LLM from scratch can cost **tens of millions of dollars**. The dominant approach is to start with **someone else's pre-trained model**.

| Model Access Type | Example |
|---|---|
| **Open Weights** | Meta's LLaMA family |
| **API Access** | OpenAI's GPT series |

**Fine-tuning** continues training a pre-trained model on a smaller, task-specific dataset:

```
Pre-trained Base Model (general knowledge)
         +
   Task-Specific Data (e.g., medical, legal)
         =
   Fine-Tuned Model (specialised behaviour)
```

| What Fine-Tuning Changes | What Fine-Tuning Does NOT Change |
|---|---|
| Task-specific behaviour | Base model weights from pre-training |
| Tone and domain knowledge | Data the fine-tuner never saw or audited |

---

### The Inheritance Problem

> When you fine-tune a pre-trained model, **you inherit everything that model already contains** — including things you cannot see and did not choose.

**Three concrete security consequences:**

**1. Safety Alignment Erodes (Doesn't Break)**

Stanford and Princeton found that safety mechanisms of aligned LLMs can be compromised by fine-tuning on **as few as 10 adversarially crafted examples** at a cost of under **$0.20**. Even benign fine-tuning on legitimate data degrades safety as a side effect.

```
Think of it like a well-worn path in a forest:
Fine-tuning adds new paths → original "safe path" gets obscured
The model hasn't forgotten how to be unsafe — the probability weights just shifted.
```

**2. Fine-Tuned Models Are More Susceptible to Prompt Injection**

Cisco found fine-tuned models are **measurably more vulnerable** to prompt injection than their base counterparts. Specialisation narrows focus and reduces resilience to unexpected tokens.

**3. Version Is Rarely Tracked**

Fine-tuning always targets a **specific checkpoint** of a base model. If that checkpoint contained a backdoor or problematic training data, **every derivative model inherits it** — regardless of whether downstream users were informed.

---

## 🔲 Model Opacity: The Black Box

Trained model weights are **billions of floating-point numbers** — the cumulative result of a training process. They carry:

- ❌ No human-readable record of how they were shaped
- ❌ No documentation of what data influenced them
- ❌ No map of what behaviours they encode

> **You cannot open a model and find the decision that makes it behave a certain way.**

Security testing of models is **sampling, not auditing**:

```
✅ Can tell you: how the model behaved on inputs you tried
❌ Cannot tell you: what it will do on inputs you haven't thought of yet
```

---

### Model Cards

A **model card** is a structured document accompanying a model that describes what it is, how it was built, and where it falls short. Introduced by Google researchers in 2019, it is the closest thing the industry has to a standard transparency format.

Think of it like a **nutritional label** for an AI model:

| Section | What It Should Tell You |
|---|---|
| **Training Data** | Sources used, filtering applied, known gaps or biases |
| **Intended Use** | What the model was designed for (and explicitly what it wasn't) |
| **Evaluation Results** | Performance metrics across different conditions |
| **Known Limitations** | Conditions where the model underperforms or behaves unexpectedly |
| **Bias Assessment** | Where training data may have introduced skew |
| **Licence** | What you're legally permitted to do with the model |

---

### The Gaps in Documentation

| Reality | Security Implication |
|---|---|
| Model cards are **voluntary** for most use cases | No regulatory requirement to produce one |
| Frequently **incomplete or vague** | Weak incentive to disclose limitations that might reduce adoption |
| Sometimes **entirely absent** | Downstream user has no audit trail |

> 🔐 **A sparse or missing model card is a warning sign** — not an inconvenience. It means the organisation either didn't evaluate the model thoroughly or chose not to share what they found.
>
> *"In security, hope is not a control."*

---

## 🎯 Key Takeaways

```
1. Training data comes from poorly documented, unaudited sources —
   most organisations cannot answer where their data came from.

2. PII and live credentials routinely get baked into model weights through
   web scraping and cannot be patched out post-deployment.

3. Model-building decisions (quantisation, federated learning) introduce
   security trade-offs that are rarely documented.

4. Fine-tuning inherits everything beneath it — safety alignment erodes
   with as few as 10 adversarial examples.

5. Model weights are fundamentally opaque — security testing samples
   behaviour rather than audits it.

6. Model cards remain voluntary, frequently incomplete, and sometimes absent.
```

---

## 📖 Quick Reference: Key Terms

| Term | Definition |
|---|---|
| **Data Provenance** | Ability to answer where data came from, when collected, and if modified |
| **Common Crawl** | The most widely used public training corpus; underpins essentially every major model |
| **ML-BOM** | Machine Learning Bill of Materials — AI equivalent of a Software SBOM |
| **Overfitting** | Model memorises training data instead of learning general patterns |
| **Epoch** | One complete pass of the training algorithm through the entire dataset |
| **Quantisation** | Reducing numerical precision of weights to improve efficiency |
| **Pruning** | Removing low-contribution parameters to shrink model size |
| **Federated Learning** | Training across decentralised devices; only weight updates (not raw data) are shared |
| **Pre-Trained Model** | A model already trained on a large general-purpose dataset |
| **Fine-Tuning** | Continuing to train a pre-trained model on a smaller, task-specific dataset |
| **Model Weights** | Billions of floating-point numbers representing the trained model's learned behaviour |
| **Model Card** | Documentation artifact describing what a model is, how it was built, and its limitations |
| **Black Box** | A model whose internal decision-making process cannot be directly inspected |

---

> 📌 *The data supply chain is as real and as exploitable as a software supply chain. For most organisations today, it is almost entirely invisible.*
