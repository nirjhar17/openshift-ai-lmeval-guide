# Evaluating AI Models on Red Hat OpenShift AI 3.0
## A Practical Guide to LM-Eval

---

## What Are We Trying to Do?

We've deployed an AI model. Great! But how do we know if it's actually good?

That's where **evaluation** comes in. We need to test our models against standardized benchmarks to answer questions like:
- How accurate is our model on science questions?
- Can it solve math problems?
- Does it generate truthful responses?
- How does it compare to other models?

Red Hat OpenShift AI provides built-in tools to answer these questions. This guide walks us through using **LM-Eval** to evaluate our language models.

---

## The Three Evaluation Tools in OpenShift AI

OpenShift AI's **TrustyAI Operator** provides three evaluation tools:

**LM-Eval**
- What It Evaluates: Language Models (LLMs)
- Best For: Testing model accuracy, reasoning, truthfulness

**RAGAS**
- What It Evaluates: RAG Systems
- Best For: Testing if our retrieval + generation pipeline works well

**Llama Stack**
- What It Evaluates: LLMs + Safety
- Best For: Advanced evaluations with guardrails (PII detection, etc.)

### What Powers LM-Eval?

LM-Eval is built on two powerful open-source projects:

**LM Evaluation Harness** (by EleutherAI)
- The core evaluation framework that runs benchmarks

**Unitxt** (by IBM)
- Provides datasets, templates, and task definitions

**This guide focuses on LM-Eval** - the most common tool for evaluating standalone language models.

---

## Understanding the Key Concepts

### The Four Key Concepts

#### 1. Card (Data Source & Preparation)

A **Card** defines where the data comes from and how to prepare it for evaluation.

```
Raw Dataset (HuggingFace)          After Card Processing
─────────────────────────          ────────────────────────
{                                  {
  "sentence1": "The cat sat..."      "text_a": "The cat sat..."
  "sentence2": "A feline was..."     "text_b": "A feline was..."
  "label": 1                         "label": "entailment"
}                                  }

Card renamed fields and converted label from number to text.
```

#### 2. Template (Prompt Formatting)

A **Template** defines how to present the question to the model.

```
Data After Card              After Template
────────────────             ──────────────────────────────────────
{                            "Given the premise: The cat sat on the mat.
  "text_a": "The cat..."      Does it entail the hypothesis: A feline was...
  "text_b": "A feline..."     
  "label": "entailment"       Choose from: entailment, not entailment
}                            
                              Answer: "
                              
Template formatted the data into a prompt the model can understand.
```

#### 3. Task (Evaluation Type)

A **Task** defines what type of evaluation this is:
- Classification (pick one answer from choices)
- Question Answering (generate an answer)
- Summarization (condense text)

#### 4. Pre-built vs Custom

**Pre-built:**
- Ready-to-use benchmarks
- Use `taskNames: arc_easy`
- Covers most use cases

**Custom:**
- Define our own evaluation
- Write card + template JSON
- For specialized needs

### How They All Connect

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│                        LMEvalJob                                    │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                      │
│         │                 │                 │                      │
│         ▼                 ▼                 ▼                      │
│      MODEL            TASK LIST         SETTINGS                   │
│   (what to test)    (how to test)    (limits, batch)              │
│                          │                                         │
│              ┌───────────┴───────────┐                            │
│              │                       │                            │
│              ▼                       ▼                            │
│         Pre-built               Custom (Unitxt)                   │
│         taskNames:              taskRecipes:                      │
│         - arc_easy                - card: (data source)           │
│         - mmlu                    - template: (prompt format)     │
│         - gsm8k                   - task: (eval type)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The Security Model (Two Levels)

OpenShift AI has a two-level security system for evaluation jobs.

**Level 1: Global (Admin Controls)** - Defines what users CAN request

```yaml
# DataScienceCluster
spec:
  components:
    trustyai:
      eval:
        lmeval:
          permitOnline: allow/deny
          permitCodeExecution: allow/deny
```

**Level 2: Per-Job (User Controls)** - Defines what THIS job needs

```yaml
# LMEvalJob
spec:
  allowOnline: true/false
  allowCodeExecution: true/false
```

**Both must be enabled for the feature to work.**

**Example combinations:**
- Global `permitOnline: allow` + Per-Job `allowOnline: true` → ✅ Works
- Global `permitOnline: allow` + Per-Job `allowOnline: false` → ❌ No internet
- Global `permitOnline: deny` + Per-Job `allowOnline: true` → ❌ Blocked by admin

---

## Part 1: Setting Up

**Prerequisites:** We expect Red Hat OpenShift AI Operator is already installed and configured on OpenShift.

### Step 1: Verify the Cluster

```bash
# Login
oc login -u cluster-admin -p <password> https://api.<cluster>:443

# Check DataScienceCluster is healthy
oc get datasciencecluster
# Should show: default-dsc   True
```

### Step 2: Enable Security Settings

Edit the DataScienceCluster to allow online access:

```yaml
spec:
  components:
    trustyai:
      managementState: Managed
      eval:
        lmeval:
          permitOnline: allow
          permitCodeExecution: allow
```

Verify the ConfigMap was updated:

```bash
oc get configmap trustyai-dsc-config -n redhat-ods-applications -o jsonpath='{.data}'
# Should show: permitOnline and permitCodeExecution as true
```

### Step 3: Create a Test Namespace

```bash
oc new-project lmeval-testing
```

### Step 4: Create HuggingFace Token Secret (if needed)

```bash
oc create secret generic hf-token-secret \
  --from-literal=token=hf_your_token_here \
  -n lmeval-testing
```

---

## Part 2: First Evaluation (HuggingFace Model)

### What Happens When We Run an Evaluation?

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   1. We submit LMEvalJob YAML                                      │
│                     │                                               │
│                     ▼                                               │
│   2. TrustyAI Operator creates a Pod with:                         │
│      ├── driver container (orchestrates)                           │
│      └── job container (runs evaluation)                           │
│                     │                                               │
│                     ▼                                               │
│   3. Job container downloads:                                      │
│      ├── Model weights from HuggingFace                            │
│      └── Dataset from HuggingFace                                  │
│                     │                                               │
│                     ▼                                               │
│   4. Runs evaluation (model answers questions)                     │
│                     │                                               │
│                     ▼                                               │
│   5. Calculates metrics (accuracy, F1, etc.)                       │
│                     │                                               │
│                     ▼                                               │
│   6. Stores results in LMEvalJob status                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### The YAML

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: LMEvalJob
metadata:
  name: my-first-eval
  namespace: lmeval-testing
spec:
  allowOnline: true
  allowCodeExecution: true
  
  # What model to test
  model: hf
  modelArgs:
    - name: pretrained
      value: google/flan-t5-small
  
  # What tests to run (pre-built)
  taskList:
    taskNames:
      - arc_easy    # Grade-school science questions
  
  limit: "10"       # Only test 10 samples (for speed)
  logSamples: true
```

### Run It

```bash
# Apply
oc apply -f my-first-eval.yaml

# Watch progress
oc get lmevaljob my-first-eval -n lmeval-testing -w

# Get results when complete
oc get lmevaljob my-first-eval -o jsonpath='{.status.results}' | jq '.results'
```

### Understanding the Results

```json
{
  "arc_easy": {
    "acc,none": 0.6,
    "acc_norm,none": 0.4
  }
}
```

**Metrics explained:**
- `acc` (60%): Model got 6 out of 10 questions right
- `acc_norm` (40%): Normalized accuracy (adjusts for answer length)

---

## Part 3: Model Types

### The Five Model Types

**hf**
- When to Use: Download model from HuggingFace
- Example: Testing open-source models

**local-completions**
- When to Use: Our deployed model (completion API)
- Example: vLLM, KServe models

**local-chat-completions**
- When to Use: Our deployed model (chat API)
- Example: Chat-optimized models

**openai-completions**
- When to Use: OpenAI API (legacy)
- Example: davinci-002, babbage-002

**openai-chat-completions**
- When to Use: OpenAI API (chat)
- Example: gpt-3.5-turbo, gpt-4

### Completions vs Chat-Completions

**Completions API** (`/v1/completions`): Takes a text prompt and continues it. Supports both loglikelihood-based tasks (multiple choice) and generation tasks.

**Chat-Completions API** (`/v1/chat/completions`): Takes a conversation with roles (system, user, assistant). Only supports generation tasks - cannot calculate loglikelihood for multiple choice questions.

---

## Part 4: Testing a Deployed Model

If we have a model running on OpenShift (via KServe/vLLM), here's how to test it.

### Finding Our Model's URL

```bash
# List services in the gateway namespace
oc get svc -n openshift-ingress | grep openshift-ai-inference

# Build the internal URL:
# http://<gateway-svc>.<namespace>.svc.cluster.local:80/<model-ns>/<model-name>/v1/completions
```

### The YAML

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: LMEvalJob
metadata:
  name: local-model-eval
  namespace: lmeval-testing
spec:
  allowOnline: true
  allowCodeExecution: true
  
  model: local-completions
  modelArgs:
    - name: model
      value: "Qwen/Qwen3-0.6B"    # Must match what server reports
    - name: base_url
      value: "http://openshift-ai-inference-openshift-ai-inference.openshift-ingress.svc.cluster.local:80/my-first-model/qwen3-0-6b/v1/completions"
    - name: tokenizer
      value: Qwen/Qwen2.5-0.5B    # For token counting
    - name: num_concurrent
      value: "1"
    - name: max_retries
      value: "3"
    - name: tokenized_requests
      value: "False"
  
  taskList:
    taskNames:
      - arc_easy
  
  limit: "10"
  batchSize: "1"
  logSamples: true
```

---

## Part 5: Understanding Task Types

### Two Types of Evaluation Tasks

**Loglikelihood Tasks**
- How It Works: Model calculates probability of each answer option
- Example Tasks: arc_easy, mmlu, hellaswag

**Generation Tasks**
- How It Works: Model writes free-form text answer
- Example Tasks: gsm8k, truthfulqa_gen

### Model Type and Task Compatibility

**hf**
- Loglikelihood Tasks: ✅ Supported
- Generation Tasks: ✅ Supported
- Notes: Full support for all task types

**local-completions**
- Loglikelihood Tasks: ✅ Supported
- Generation Tasks: ✅ Supported
- Notes: Full support for all task types

**local-chat-completions**
- Loglikelihood Tasks: ❌ Not Supported
- Generation Tasks: ✅ Supported
- Notes: Chat API cannot return token probabilities

**openai-completions**
- Loglikelihood Tasks: ⚠️ Limited (only davinci-002 and babbage-002)
- Generation Tasks: ✅ Supported

**openai-chat-completions**
- Loglikelihood Tasks: ❌ Not Supported
- Generation Tasks: ✅ Supported
- Notes: Chat API cannot return token probabilities

**Recommendation:** Use `local-completions` or `hf` for maximum flexibility with all task types.

### Popular Tasks Reference

**Loglikelihood Tasks:**
- `arc_easy` - Grade-school science (easy)
- `arc_challenge` - Grade-school science (hard)
- `mmlu` - 57 subjects from elementary to professional
- `hellaswag` - Common sense reasoning
- `winogrande` - Pronoun resolution

**Generation Tasks:**
- `gsm8k` - Grade school math
- `truthfulqa_gen` - Truthfulness evaluation
- `drop` - Reading comprehension

---

## Part 6: Testing with OpenAI

### Create the OpenAI Secret

```bash
oc create secret generic openai-api-secret \
  --from-literal=OPENAI_API_KEY=sk-your-key-here \
  -n lmeval-testing
```

### OpenAI Completions (Legacy Models)

For loglikelihood tasks, use legacy models:

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: LMEvalJob
metadata:
  name: openai-completions-eval
  namespace: lmeval-testing
spec:
  allowOnline: true
  allowCodeExecution: true
  model: openai-completions
  modelArgs:
    - name: model
      value: "davinci-002"    # or babbage-002
  taskList:
    taskNames:
      - arc_easy
  limit: "5"
  batchSize: "1"
  logSamples: true
  pod:
    container:
      env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: openai-api-secret
              key: OPENAI_API_KEY
```

### OpenAI Chat (GPT-3.5/4)

For generation tasks with modern models:

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: LMEvalJob
metadata:
  name: openai-chat-eval
  namespace: lmeval-testing
spec:
  allowOnline: true
  allowCodeExecution: true
  model: openai-chat-completions
  modelArgs:
    - name: model
      value: "gpt-3.5-turbo"
  chatTemplate:
    enabled: true              # Required for chat models
  taskList:
    taskNames:
      - gsm8k                  # Generation task
      - truthfulqa_gen         # Generation task
  limit: "5"
  batchSize: "1"
  logSamples: true
  pod:
    container:
      env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: openai-api-secret
              key: OPENAI_API_KEY
```

---

## Part 7: Custom Evaluations

When pre-built tasks don't fit our needs, we can create custom ones.

### When to Use Custom

**Use Pre-built when:**
- Standard benchmarks needed
- Quick testing
- Most use cases

**Use Custom when:**
- Our own dataset
- Specific prompt format
- Domain-specific evaluation

### Working Custom Evaluation Example

Here's a complete working example using a local deployed model with a custom Unitxt card:

```yaml
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: LMEvalJob
metadata:
  name: qwen-custom-test
  namespace: lmeval-testing
spec:
  allowOnline: true
  allowCodeExecution: true
  model: local-completions
  modelArgs:
    - name: model
      value: "Qwen/Qwen3-0.6B"
    - name: base_url
      value: "http://openshift-ai-inference-openshift-ai-inference.openshift-ingress.svc.cluster.local:80/my-first-model/qwen3-0-6b/v1/completions"
    - name: tokenizer
      value: Qwen/Qwen2.5-0.5B
    - name: num_concurrent
      value: "1"
    - name: max_retries
      value: "3"
    - name: tokenized_requests
      value: "False"
  taskList:
    taskRecipes:
      - template:
          name: "templates.classification.multi_class.relation.default"
        card:
          custom: |
            {
              "__type__": "task_card",
              "loader": {
                "__type__": "load_hf",
                "path": "glue",
                "name": "wnli"
              },
              "preprocess_steps": [
                {"__type__": "rename", "field": "sentence1", "to_field": "text_a"},
                {"__type__": "rename", "field": "sentence2", "to_field": "text_b"},
                {"__type__": "map_instance_values", "mappers": {"label": {"0": "not entailment", "1": "entailment", "-1": "unknown"}}},
                {"__type__": "set", "fields": {"classes": ["entailment", "not entailment", "unknown"], "text_a_type": "premise", "text_b_type": "hypothesis", "type_of_relation": "entailment"}}
              ],
              "task": "tasks.classification.multi_class.relation"
            }
        numDemos: 3
        demosPoolSize: 50
  logSamples: true
  limit: "10"
  batchSize: "1"
```

### Understanding the Custom Card

**loader** - Defines where to load data from (HuggingFace glue/wnli)

**preprocess_steps** - Transforms the data fields to match task requirements

**rename** - Changes field names (sentence1 → text_a)

**map_instance_values** - Converts label values (0 → "not entailment")

**set** - Adds required fields (classes, type_of_relation)

**task** - Specifies the evaluation type

### Custom Template Example

We can also define a custom template to control exactly how the prompt is formatted:

```yaml
template:
  custom: |
    {
      "__type__": "input_output_template",
      "instruction": "Determine if the hypothesis follows from the premise. Answer only 'yes' or 'no'.",
      "input_format": "Premise: {text_a}\nHypothesis: {text_b}",
      "output_format": "{label}",
      "target_prefix": "Answer: "
    }
```

This generates prompts like:

```
Determine if the hypothesis follows from the premise. Answer only 'yes' or 'no'.

Premise: The cat sat on the mat.
Hypothesis: A feline was on a floor covering.

Answer: 
```

### Key Learning: Model Choice Matters!

Custom evaluations use generation (model must output exact text). Not all models handle this well:

**flan-t5-base (HF):** 0% accuracy

**Qwen3-0.6B (local):** 50% accuracy, 67% F1

Models trained with better instruction-following capabilities perform significantly better on custom evaluations.

---

## Results Summary

Here's what we achieved across our tests:

**Part 2: flan-t5-small (HF)**
- Task: wnli
- Score: 56% accuracy

**Part 4: Qwen3-0.6B (local)**
- Task: arc_easy
- Score: 60% accuracy

**Part 4: Qwen3-0.6B (chat)**
- Task: gsm8k
- Score: 20% exact match

**Part 6: davinci-002 (OpenAI)**
- Task: arc_easy
- Score: 40% accuracy

**Part 6: gpt-3.5-turbo (OpenAI)**
- Task: gsm8k
- Score: 80% exact match

**Part 7: flan-t5-base (HF)**
- Task: pre-built wnli
- Score: 60% accuracy

**Part 7: Qwen3-0.6B (local)**
- Task: custom wnli
- Score: 50% accuracy / 67% F1

**Key Insights:** 
- GPT-3.5-turbo significantly outperformed smaller models on reasoning tasks.
- Custom evaluations work well with models that have strong instruction-following capabilities.
- The same task (wnli) can yield different results depending on whether we use pre-built (loglikelihood) vs custom (generation) evaluation.

---

## Part 8: Dashboard UI

OpenShift AI provides a visual interface for running evaluations.

### Enabling the Dashboard

Set `disableLMEval: false` in the `OdhDashboardConfig` CR.

### Using the Dashboard

1. Navigate to **Evaluations** in the OpenShift AI Dashboard
2. Select the project
3. Click **Start evaluation run**
4. Fill in the form:
   - Model name
   - Evaluation tasks
   - Security settings
5. Click **Evaluate**

---

## Useful Commands

```bash
# Check job status
oc get lmevaljob <name> -n lmeval-testing

# Get results
oc get lmevaljob <name> -o jsonpath='{.status.results}' | jq

# View logs
oc logs <pod-name> -n lmeval-testing

# Check events
oc get events -n lmeval-testing --sort-by='.lastTimestamp'
```

---

## What's Next?

This guide covered the core LM-Eval functionality. For advanced topics, see:

- **LLM-as-a-Judge** - Using one LLM to evaluate another (separate blog)
- **RAGAS** - Evaluating RAG systems
- **Llama Stack** - Advanced evaluations with guardrails

---

## Resources

- [Red Hat OpenShift AI Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html-single/evaluating_ai_systems/index)
- [LM Evaluation Harness (EleutherAI)](https://github.com/EleutherAI/lm-evaluation-harness)
- [Unitxt](https://github.com/IBM/unitxt)
