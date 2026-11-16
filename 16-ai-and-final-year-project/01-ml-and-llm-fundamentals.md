# ML and LLM Fundamentals

> **Tier:** T2 (working knowledge) · **Related:** [Human activity recognition project](02-human-activity-recognition-project.md), [Python fundamentals](../03-languages/python/01-fundamentals.md), [PostgreSQL](../12-databases/04-postgresql.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md)
> **On your resume:** Final Year Project "Recognition and Demonstrations of Human Activities Leveraging LLMs" (.NET, Python), built with fine-tuned LLMs. This file covers the theory behind it, plus what full-stack interviews ask about shipping LLM features.

## Quick revision
- **Supervised** learning uses labeled examples (classification, regression); **unsupervised** finds structure without labels (clustering, dimensionality reduction); **self-supervised** creates labels from the data itself (next-token prediction), which is how LLMs are pretrained.
- **Classification** predicts a category; **regression** predicts a number.
- **Splits:** train to fit, validation to tune and choose, test once at the end. Fit preprocessing (scalers, vocabularies) on the training data only.
- **Overfitting:** excellent training score, poor validation score. Fixes: more data, a simpler model, regularization (L1, L2, dropout), early stopping, augmentation.
- **Cross-validation:** k folds, average the scores; stratified to keep class balance; grouped (`GroupKFold`) when samples from one person or session must stay together.
- **Metrics:** precision = TP / (TP + FP); recall = TP / (TP + FN); F1 = their harmonic mean; the confusion matrix shows which classes get confused. **Macro** averages per-class scores equally; **micro** pools all decisions (equal to accuracy in single-label multiclass); **weighted** weights classes by support. Accuracy misleads on imbalanced data.
- **Neural networks:** layers of weighted sums plus nonlinear activations, trained by backpropagation and gradient descent on a loss (cross-entropy for classification).
- **Sequences:** 1D CNNs find local patterns, RNNs, LSTMs and GRUs carry state through time, transformers use attention and parallelize well.
- **Attention:** each token's new representation is a weighted mix of all tokens' values, with weights from query-key similarity: softmax(QK^T / sqrt(d_k)) V.
- **Tokens** are subword pieces, **embeddings** are vectors where similar meanings sit close together, the **context window** caps prompt plus output tokens.
- **LLM training stages:** pretraining (next-token prediction) -> instruction tuning (supervised fine-tuning on instruction-response pairs) -> preference tuning (RLHF or DPO).
- **Prompting vs RAG vs fine-tuning:** prompt first; add RAG when the model needs your data or fresh facts; fine-tune for consistent behavior or format, or to make a small model good at a narrow task.
- **LoRA** trains small low-rank adapter matrices while base weights stay frozen; **QLoRA** does this on a 4-bit quantized base model.
- **Hallucination:** fluent, confident, wrong. Mitigate with grounding and citations, constrained outputs, validation, evaluation sets and an explicit "unknown" path.
- **Shipping:** call models from the backend (keys never in clients), stream for UX, validate structured output, authorize tool calls, budget tokens, cache, redact logs, handle timeouts.
- **RAG:** chunk -> embed -> store (for example pgvector) -> retrieve top-k -> generate with citations.

## 1. Learning types and tasks

| Type | Data | Examples |
|---|---|---|
| Supervised | Inputs with labels | Activity from sensor window, spam or not, house price |
| Unsupervised | Inputs only | Clustering users, anomaly detection, PCA |
| Self-supervised | Labels derived from the data | Next-token prediction, masked words, contrastive learning on sensor data |
| Reinforcement | Actions and rewards | Game agents, the RL step of RLHF |

Classification can be **binary** (fall or no fall), **multiclass** (walking, sitting, running: exactly one label) or **multilabel** (several labels at once). Regression outputs a continuous value (calories burned, step length).

## 2. Data splits and leakage
The test set estimates performance on unseen data, so it must stay unseen: use it once, at the end. The validation set (or cross-validation) is where you tune hyperparameters and pick models.

**Leakage** is information from the evaluation data sneaking into training, producing scores that collapse in real use:
- Fitting a scaler or normalizer on the whole dataset before splitting.
- The same person (or session) in both train and test, so the model learns the person, not the activity.
- Overlapping time windows split randomly: neighbors share most of their samples.
- Duplicates across splits, or future information in time-series features.

```python
from sklearn.model_selection import GroupShuffleSplit

# X: features per window, y: labels, groups: the subject each window came from
splitter = GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=42)
train_idx, test_idx = next(splitter.split(X, y, groups=groups))   # no subject in both sides
```

## 3. Overfitting and underfitting

| | Underfitting | Overfitting |
|---|---|---|
| Symptom | Poor on train and validation | Great on train, poor on validation |
| Cause | Model too simple, weak features, too little training | Model too complex for the data, noise memorized |
| Fixes | Bigger model, better features, train longer | More data, augmentation, regularization (L1, L2, dropout), early stopping, simpler model |

The bias-variance intuition: simple models are biased (systematically wrong), complex ones have high variance (sensitive to the particular training set). Validation curves show where you are.

## 4. Cross-validation
- **k-fold:** split into k parts, train on k - 1, validate on the remaining one, rotate, average.
- **Stratified k-fold:** each fold keeps the class proportions; important for imbalanced classes.
- **Group k-fold:** all samples of a group (subject, session, device) stay in one fold; the right choice for human activity data.
- **Time-series split:** train on the past, validate on the future.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import GroupKFold, cross_val_score

scores = cross_val_score(
    RandomForestClassifier(n_estimators=200, random_state=42),
    X, y, groups=groups, cv=GroupKFold(n_splits=5), scoring="f1_macro",
)
print(f"macro F1 {scores.mean():.3f} +/- {scores.std():.3f}")
```

Leave-one-subject-out is group k-fold with one subject per fold (`LeaveOneGroupOut` in scikit-learn).

## 5. Classification metrics, worked by hand
Six windows, three activities:

```python
from sklearn.metrics import classification_report, confusion_matrix, f1_score

y_true = ["walk", "walk", "sit", "run", "sit", "run"]
y_pred = ["walk", "sit", "sit", "run", "sit", "walk"]
labels = ["walk", "sit", "run"]

print(confusion_matrix(y_true, y_pred, labels=labels))   # rows = true, columns = predicted
print(f1_score(y_true, y_pred, average="macro"))          # about 0.66
print(classification_report(y_true, y_pred, labels=labels))
```

Confusion matrix (rows true, columns predicted):

| True \ Predicted | walk | sit | run |
|---|---|---|---|
| walk | 1 | 1 | 0 |
| sit | 0 | 2 | 0 |
| run | 1 | 0 | 1 |

| Class | Precision | Recall | F1 |
|---|---|---|---|
| walk | 1/2 = 0.50 | 1/2 = 0.50 | 0.50 |
| sit | 2/3 = 0.67 | 2/2 = 1.00 | 0.80 |
| run | 1/1 = 1.00 | 1/2 = 0.50 | 0.67 |

Accuracy = 4/6 = 0.67. Macro F1 = (0.50 + 0.80 + 0.67) / 3 = 0.66. Micro F1 = 0.67, the same as accuracy. The table tells a story accuracy hides: "run" is often missed (recall 0.50) and "sit" absorbs other classes (precision 0.67).

**Precision or recall?** Prefer precision when false alarms are costly (flagging fraud that is not fraud); prefer recall when misses are costly (missing a fall in a fall-detection feature).

## 6. Neural networks in one page
- A neuron computes `activation(w . x + b)`; layers stack neurons. Activations add nonlinearity: ReLU in hidden layers, sigmoid for binary outputs, softmax to turn scores into class probabilities.
- A **loss** measures error: cross-entropy for classification, mean squared error for regression.
- **Training:** a forward pass computes predictions and the loss; **backpropagation** computes the gradient of the loss for every weight; an **optimizer** (SGD, Adam) nudges the weights. One pass over the data is an **epoch**; data is processed in **batches**; the **learning rate** controls step size.
- **For time series such as sensor data:**

| Architecture | Idea | Strength | Weakness |
|---|---|---|---|
| 1D CNN | Filters slide along time and detect local motion patterns | Fast, strong baseline | Limited long-range context |
| RNN | A hidden state updated at each step | Natural for sequences | Vanishing gradients, slow |
| LSTM / GRU | Gates decide what to keep or forget | Longer dependencies | Still sequential, slower to train |
| CNN-LSTM | CNN features fed into an LSTM | Local plus temporal patterns | More tuning |
| Transformer | Attention over all time steps | Parallel, long context | Data-hungry, cost grows with sequence length |

## 7. Transformers and attention
For every token, the model computes a **query** (what am I looking for?), a **key** (what do I contain?) and a **value** (what do I pass on?). Attention weights come from query-key similarity, normalized with softmax; each token's output is the weighted sum of values: `Attention(Q, K, V) = softmax(Q K^T / sqrt(d_k)) V`. **Multi-head** attention runs several of these in parallel to capture different relationships. Attention alone ignores order, so **positional encodings** are added. Cost grows quadratically with sequence length, which is why context windows are limited.

| Family | Masking | Typical use |
|---|---|---|
| Encoder (BERT-style) | Sees the whole input | Classification, embeddings, search |
| Decoder (GPT-style) | Causal: sees only earlier tokens | Text generation, chat models |
| Encoder-decoder (T5-style) | Encoder reads, decoder writes | Translation, summarization |

## 8. Tokens, embeddings, context windows and sampling
- **Tokens:** text is split into subword pieces by a tokenizer (byte-pair encoding and similar). A common rule of thumb for English is about four characters per token, but it varies by tokenizer. Numbers often split into several tokens, which matters when you feed sensor values to an LLM.
- **Embeddings:** dense vectors; texts with similar meaning have high cosine similarity. They power semantic search, clustering and RAG.
- **Context window:** the maximum tokens per request, prompt plus output. Longer prompts cost more and are slower, and models can overlook details buried in very long contexts.
- **Sampling:** temperature controls randomness (low for extraction and classification, higher for creative text); top-p limits sampling to the most likely tokens. Even at temperature 0, outputs are not guaranteed to be identical across runs.

## 9. How LLMs are trained
1. **Pretraining:** next-token prediction over huge text corpora; the model learns language, facts and patterns. Most of the compute goes here. Result: a **base model** that continues text.
2. **Instruction tuning (supervised fine-tuning):** curated prompt-response pairs teach it to follow instructions in a chat format.
3. **Preference tuning:** people (or models) rank candidate answers. **RLHF** trains a reward model on those rankings and optimizes the LLM against it with reinforcement learning; **DPO** optimizes directly on preference pairs without a separate reward model.

Your FYP's fine-tuning is a fourth, task-specific step on top of a pretrained (often instruction-tuned) model.

## 10. Prompting vs RAG vs fine-tuning

| Need | Prompting | RAG | Fine-tuning |
|---|---|---|---|
| The model lacks your private or fresh data | No | Yes | Poor fit: knowledge goes stale and is hard to update |
| A consistent output format or style | Often enough, with examples and structured output | No | Yes |
| A narrow task at high volume, low latency or cost | Possible with a large model | No | Yes: a small fine-tuned model |
| Cost to start | Lowest | Medium (a retrieval pipeline) | Highest (data, training, evaluation) |
| Updating knowledge | Edit the prompt | Re-index documents | Retrain |
| Showing sources | Hard | Natural (citations) | Hard |

Default path: prompting plus an evaluation set; add RAG for knowledge; fine-tune for behavior or efficiency once you have data and a baseline to beat.

## 11. LoRA and QLoRA
Full fine-tuning updates every weight and needs memory for the weights, their gradients and the optimizer state. **LoRA** freezes a weight matrix W (d x k) and learns a low-rank update `delta W = B A`, with B (d x r) and A (r x k) and a small rank r such as 8 or 16; the layer computes `W x + (alpha / r) B A x`. Trainable parameters drop by orders of magnitude, adapters are small files you can swap per task, and they can be merged into the base weights for inference. **QLoRA** loads the base model in 4-bit precision (the NF4 data type) and trains LoRA adapters on top, which brings fine-tuning of larger models within reach of a single GPU at a small quality cost.

```python
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained("base-model-id")   # placeholder model id
config = LoraConfig(
    r=8,
    lora_alpha=16,
    lora_dropout=0.05,
    target_modules=["q_proj", "v_proj"],   # attention projections; names depend on the architecture
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, config)
model.print_trainable_parameters()          # typically well under 1% of all parameters
```

## 12. Evaluating LLM output and hallucinations
- **When the output is a label** (as in activity recognition), parse it and use ordinary metrics: accuracy, macro F1, per-class F1, a confusion matrix. Count unparseable or out-of-vocabulary answers as errors.
- **When the output is free text:** reference-based metrics (exact match, token F1, BLEU or ROUGE) capture overlap but not correctness; human rating with a rubric is the gold standard; using an LLM as a judge scales but has biases, so check it against human ratings.
- **Keep a fixed evaluation set** and rerun it whenever the prompt, model or data changes; treat it like a regression test suite.
- **Hallucinations** happen because a model generates plausible continuations, not verified facts. Reduce them with grounding (RAG with citations), constrained outputs (a fixed label set, JSON schemas), lower temperature for factual tasks, instructions and an output path for "unknown", and validation of anything that feeds code or users.

## 13. Shipping LLM features in apps
- **Architecture:** client -> your backend -> model provider (or a self-hosted model). API keys live only on the server; never in a web bundle or a mobile app.
- **Streaming:** send tokens as they arrive (server-sent events or a chunked response) so users see progress; support cancellation.
- **Structured output:** request JSON that matches a schema (many providers offer a JSON or schema mode; check yours), then validate it anyway and retry or fall back when it is invalid.
- **Tool calling:** the model proposes a tool name and arguments; your code validates them, checks the user's permissions, executes and returns the result. The model must never be able to do more than the user could.
- **Cost and latency:** you pay per input and output token. Choose the smallest model that passes your evaluation, trim prompts, cap output length, cache, and move bulk work into background jobs.
- **Caching:** exact-match caches on normalized inputs; some providers also discount repeated prompt prefixes (check their docs).
- **Privacy and safety:** send only the data needed, check the provider's retention and training policies, respect client NDAs, defend against prompt injection (untrusted text in the context can contain instructions, so treat model output as untrusted input), rate-limit per user, and log with personal data redacted.
- **Reliability:** timeouts, retries with backoff, fallbacks (a smaller model, a cached answer, a friendly error), and idempotency for any tool with side effects.

```ts
import { z } from "zod";

const ActivityAnswer = z.object({
  activity: z.enum(["walking", "running", "sitting", "standing", "stairs"]),
  confidence: z.number().min(0).max(1),
});

// Treat model output as untrusted input: parse, validate, and fail safely.
export function parseActivityAnswer(raw: string) {
  try {
    const result = ActivityAnswer.safeParse(JSON.parse(raw));
    return result.success ? result.data : null;
  } catch {
    return null; // not JSON at all: retry, fall back, or report "unknown"
  }
}
```

## 14. RAG basics with pgvector
1. **Ingest:** split documents into chunks (by tokens with some overlap, or by headings), keep metadata (source, title, tenant).
2. **Embed** each chunk with an embedding model; store the vector.
3. **Retrieve:** embed the question with the same model, fetch the top-k most similar chunks (optionally combine with keyword search and rerank).
4. **Generate:** put the chunks in the prompt, instruct the model to answer only from them and cite sources.
5. **Evaluate** retrieval (does the right chunk appear in the top k?) separately from answer quality (is the answer faithful to the chunks?).

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE doc_chunks (
  id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  org_id    uuid NOT NULL,              -- tenant filter: never retrieve another org's documents
  source    text NOT NULL,
  content   text NOT NULL,
  embedding vector(1536) NOT NULL        -- the dimension must match your embedding model
);
CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);

-- $1 = query embedding, $2 = current org; <=> is cosine distance (smaller is closer)
SELECT source, content
FROM doc_chunks
WHERE org_id = $2
ORDER BY embedding <=> $1
LIMIT 5;
```

pgvector also offers `<->` (Euclidean distance) and `<#>` (negative inner product), and IVFFlat indexes as an alternative to HNSW.

## Common pitfalls
- Random splits on correlated samples (same subject, overlapping windows), giving inflated scores.
- Reporting only accuracy on imbalanced classes.
- Tuning on the test set, or fitting preprocessing on all the data.
- No baseline: an LLM result means little without a simple model to compare against.
- Trusting model JSON or labels without validation.
- API keys shipped inside web or mobile clients.
- Fine-tuning to inject knowledge that changes; that is a retrieval problem.
- Ignoring token costs until the bill arrives.
- Vector search without a tenant filter in a multi-tenant app.
- Letting a model call tools with more permissions than the user has.

## How to talk about this in an interview
- **State your real exposure:** your final-year project used fine-tuned LLMs for activity recognition in a .NET and Python web app. That is academic experience; your resume shows no production LLM features, so do not imply them.
- **Answer "What AI experience do you have?"** in three parts: the FYP (what you built, which model, how you evaluated), the fundamentals you understand (this file), and how you would apply them in a product, using your production skills (background jobs, rate limits, multi-tenant scoping, API design).
- **Bridge to full-stack work:** putting a model behind an HTTP service, calling it from a web app, validating its output and handling its failures are integration problems you already solve with third-party APIs.

> **Fill in:** which base model you fine-tuned, the fine-tuning method (LoRA, QLoRA, full fine-tuning, a provider's fine-tuning service), the dataset, your metrics and results, and which parts you personally built.

> **Fill in:** any AI-related coursework or self-study (courses, papers, small projects), and how you use coding assistants at work, if at all (see [common HR questions](../18-behavioral-and-hr/03-common-hr-questions.md), section 17).

## Interview questions

**Q1. What is the difference between supervised and unsupervised learning?**
Supervised learning trains on labeled examples to predict labels for new inputs (classification, regression). Unsupervised learning finds structure without labels (clustering, dimensionality reduction). LLM pretraining is self-supervised: the next token is the label.
*Follow-up:* Is your FYP supervised? Yes: windows of sensor data labeled with activities.

**Q2. Explain precision and recall. When would you favor each?**
Precision is the share of predicted positives that are correct; recall is the share of actual positives found. Favor precision when false alarms are expensive, recall when misses are expensive (for example detecting falls).
*Follow-up:* Why F1? It balances both and drops sharply when either is low.

**Q3. Macro vs micro averaging: which would you report for activity recognition?**
Macro averages per-class F1 equally, so rare activities count as much as common ones; micro pools all decisions and equals accuracy for single-label problems. Activity datasets are usually imbalanced, so report macro F1 plus per-class F1 and a confusion matrix.

**Q4. How do you detect and prevent overfitting?**
Compare training and validation curves: a widening gap means overfitting. Prevent it with more or augmented data, regularization (L2, dropout), early stopping, a simpler model, and honest validation (grouped cross-validation).

**Q5. Why separate validation and test sets?**
Every decision made on a dataset leaks information into the model. The validation set absorbs tuning decisions; the untouched test set gives an unbiased final estimate. Tuning on the test set makes its score optimistic.

**Q6. What is cross-validation, and when do you need a grouped version?**
Rotate which fold is held out and average the scores, for a more stable estimate. Use grouped folds when samples are correlated within a group, such as windows from the same person; otherwise the model is tested on people it has already seen.

**Q7. Explain attention to a non-specialist.**
When reading a word, the model looks at every other word and decides how much each one matters for understanding it, then blends their information with those weights. Technically: query-key similarity, softmax, weighted sum of values, in several heads at once.

**Q8. What are tokens, embeddings and a context window?**
Tokens are the subword units a model reads and is billed by; embeddings are vectors that represent meaning so similar texts are close; the context window is the maximum number of tokens per request, input plus output.

**Q9. Prompting, RAG or fine-tuning: how do you choose?**
Start with prompting and an evaluation set. Use RAG when answers depend on private or changing data. Fine-tune when you need consistent behavior or format, or a smaller, cheaper model for a narrow task, and you have enough quality examples.
*Follow-up:* Why did your FYP fine-tune? Answer from your project; a typical reason is teaching the model a narrow task and output format it does not do reliably with prompting alone.

**Q10. What are LoRA and QLoRA?**
LoRA freezes the pretrained weights and trains small low-rank matrices added to selected layers, so only a tiny fraction of parameters train and adapters are small. QLoRA does the same on a base model quantized to 4 bits, cutting memory further so larger models fit on one GPU.

**Q11. What are hallucinations, and how do you reduce them?**
Confident output that is not true or not supported. Reduce them with grounding and citations, constrained output formats, lower temperature for factual tasks, an allowed "unknown" answer, validation of outputs, and evaluation sets that catch regressions.

**Q12. How would you add an LLM feature to a Rails or Next.js app safely?**
Call the provider from the backend with server-side keys; for slow or bulk work use a background job; stream responses for interactive features; validate structured output; authorize every tool call as the current user; rate-limit per user; redact personal data in logs; set timeouts, retries and fallbacks; track cost per request.

**Q13. How does RAG work, and how would you store embeddings in PostgreSQL?**
Chunk documents, embed the chunks, store vectors in a `vector` column (pgvector) with an HNSW index, embed the question, retrieve the nearest chunks filtered by tenant, and generate an answer constrained to those chunks with citations.

**Q14. What AI experience do you have?**
Answer honestly in three parts: the FYP (fine-tuned LLMs recognizing activities from smartwatch, phone and glasses data, with a .NET web app and Python), the fundamentals you can explain (metrics, leakage-free evaluation, LoRA, RAG), and how you would ship an LLM feature using your production experience. Do not claim production AI work you have not done.
> **Fill in:** two concrete details from your FYP that you would mention here.

## Further reading
- scikit-learn, model evaluation: https://scikit-learn.org/stable/modules/model_evaluation.html
- scikit-learn, cross-validation: https://scikit-learn.org/stable/modules/cross_validation.html
- Hugging Face PEFT documentation: https://huggingface.co/docs/peft
- pgvector: https://github.com/pgvector/pgvector
