# Human Activity Recognition with LLMs: Your Final-Year Project

> **Tier:** T2 (working knowledge) · **Related:** [ML and LLM fundamentals](01-ml-and-llm-fundamentals.md), [ASP.NET Core MVC](../11-dotnet/01-aspnet-core-mvc.md), [Blazor](../11-dotnet/03-blazor.md), [Python fundamentals](../03-languages/python/01-fundamentals.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md), [Tell me about yourself](../18-behavioral-and-hr/01-tell-me-about-yourself.md)
> **On your resume:** Final Year Project: Recognition and Demonstrations of Human Activities Leveraging LLMs (.NET, Python): a web app recognizing human activities from smartwatch, phone and glasses data with fine-tuned LLMs.

## Quick revision
- **What the resume states:** a web app; recognizes human activities; data from a smartwatch, a phone and glasses; fine-tuned LLMs; .NET and Python. Everything else in this file is a typical approach, labeled as such. Replace it with what your team actually did.
- **HAR** (human activity recognition) classifies what a person is doing (walking, sitting, climbing stairs) from sensor signals.
- **Sensors:** accelerometer (acceleration including gravity), gyroscope (rotation rate), magnetometer (heading), barometer (altitude change), heart rate on watches; glasses typically add a head-mounted motion sensor and, depending on the model, a camera.
- **Sampling rates** in HAR datasets are commonly between about 20 and 100 Hz; everyday movement is mostly low-frequency.
- **Multi-device sync:** a common clock, resampling to one rate, interpolation of small gaps, a policy for missing devices.
- **Windowing:** fixed-length sliding windows of a few seconds, often with 50% overlap, one label per window (the majority label).
- **Normalize** each channel with statistics from the training data only.
- **Inputs:** hand-crafted features (mean, standard deviation, RMS, dominant frequency) for classic models; raw sequences for deep models.
- **Baselines:** random forest or SVM on features; 1D CNN or LSTM on raw windows. An LLM result means little without one.
- **LLM approaches:** serialize features or raw values as text; fine-tune (often with LoRA) on prompt-to-label pairs; fuse a sensor encoder into the LLM's embedding space; or let an LLM reason over a classic classifier's outputs.
- **Leakage-free evaluation:** subject-wise splits or leave-one-subject-out; macro F1, per-class F1, a confusion matrix.
- **Typical architecture:** a .NET web app (UI, uploads, users, results) calling a Python model service over HTTP with JSON.
- **"Demonstrations"** can mean visual demonstrations of recognized activities or in-context examples in prompts; know which your title means.

> **Fill in:** your team size and your role, the base LLM, the fine-tuning method, the dataset (public or collected by you), the activity classes, your metrics and results, and the baseline you compared against.

## 1. The project in one minute
Built only from resume facts, with brackets for what only you know:

> "My final-year project was a web app that recognizes human activities from smartwatch, phone and glasses data using fine-tuned large language models, built with .NET and Python. [How data gets in: uploaded recordings or live streaming.] A Python service [what it does: synchronizes the devices, cuts the signals into windows, and asks the fine-tuned model for the activity], and the .NET web app [what users see: activity timelines, the 'demonstrations', statistics]. We fine-tuned [model] on [dataset] and measured [metrics], reaching [result] compared with [baseline]. My part was [your role]."

Practice it until it fits in 60 seconds, and keep a 20-second version for "tell me about yourself".

## 2. HAR from scratch

### Activities
Typical classes: locomotion (walking, running, stairs up and down), static postures (sitting, standing, lying), transitions (sit-to-stand), daily activities (eating, brushing teeth, typing) and events (falls). Static postures are easily confused with each other, and fine-grained daily activities are harder than locomotion.

### Sensors

| Sensor | Measures | Units | Helps with |
|---|---|---|---|
| Accelerometer | Acceleration on 3 axes, including gravity | m/s^2 or g | Movement intensity; posture from the direction of gravity |
| Gyroscope | Angular velocity on 3 axes | rad/s or deg/s | Turning, arm swing, rotations |
| Magnetometer | Magnetic field | microtesla | Heading and orientation |
| Barometer | Air pressure | hPa | Altitude change: stairs, elevators |
| Heart rate (optical, on watches) | Pulse | beats per minute | Exertion level |
| Glasses (device-dependent) | Head motion; possibly camera or eye tracking | varies | Head movement, eating, reading, looking around |

### Placement matters
A phone in a pocket, a watch on the wrist and glasses on the head see different signals: the wrist captures hand activity (eating, typing), the head captures head motion and orientation, the pocket captures leg movement. Combining devices is what lets a system separate activities that look alike to one sensor.

### Sampling rates
By the Nyquist theorem you must sample at more than twice the highest frequency you care about. Everyday movement is mostly low-frequency, so rates around 50 Hz are common (the UCI HAR smartphone dataset used 50 Hz); higher rates cost battery, bandwidth and storage.

> **Fill in:** the activities, devices (models if relevant), sensors and sampling rates in your project.

## 3. Preparing the data

### Synchronizing several devices
Problems: each device has its own clock (offset and drift), its own sampling rate, irregular intervals (jitter), dropped samples, and sometimes a device missing entirely. Typical solutions:
- Convert all timestamps to one reference clock (for example, the phone's).
- Record a sync event visible on every device (a shake or a jump) and align on it, by eye or by cross-correlation.
- Resample every channel onto one shared time grid.
- Mark gaps, and decide what happens when a device is missing: drop the window, impute, or train the model to cope.

```python
import numpy as np

def resample(timestamps_s: np.ndarray, values: np.ndarray, hz: int, start: float, end: float) -> np.ndarray:
    """Linearly interpolate one channel onto a regular grid shared by all devices.
    timestamps_s must be increasing; small gaps are bridged, so mark large ones separately."""
    grid = np.arange(start, end, 1.0 / hz)
    return np.interp(grid, timestamps_s, values)
```

### Windowing with overlap

```python
def sliding_windows(signal: np.ndarray, labels: np.ndarray, size: int, step: int):
    """signal: (n_samples, n_channels); labels: (n_samples,). Yields (window, majority_label)."""
    for start in range(0, len(signal) - size + 1, step):
        values, counts = np.unique(labels[start:start + size], return_counts=True)
        yield signal[start:start + size], values[counts.argmax()]

# At 50 Hz, 2.56 s windows are 128 samples; 50% overlap means a step of 64 (the UCI HAR setup).
```

Trade-offs: short windows react faster but carry less context; long windows capture slow activities but blur transitions; overlap adds training examples but makes neighboring windows nearly identical, which matters for evaluation (section 6).

### Normalization and filtering
- Z-score each channel using the training set's mean and standard deviation, never the full dataset's.
- Low-pass filtering removes high-frequency noise; separating gravity (low-frequency) from body motion (the rest) helps with posture vs movement.
- The magnitude `sqrt(x^2 + y^2 + z^2)` does not depend on how the device is oriented, which helps when a phone sits differently in every pocket.

### Hand-crafted features vs raw sequences

| | Hand-crafted features | Raw sequences |
|---|---|---|
| What | Statistics per window and channel: mean, standard deviation, min, max, range, RMS, median, interquartile range, zero crossings, signal magnitude area, axis correlations, dominant frequency, spectral energy, entropy | The windowed signal itself |
| Used by | Random forests, SVMs, kNN, and text prompts for LLMs (compact) | CNNs, LSTMs, transformers |
| Pros | Interpretable, small, fast | The model learns its own features |
| Cons | Feature engineering effort, may miss patterns | Needs more data and compute |

```python
def window_features(window: np.ndarray) -> dict[str, float]:
    """window: (n_samples, 3) accelerometer x, y, z from one device."""
    magnitude = np.linalg.norm(window, axis=1)                    # orientation-invariant
    spectrum = np.abs(np.fft.rfft(magnitude - magnitude.mean()))
    return {
        "mag_mean": float(magnitude.mean()),
        "mag_std": float(magnitude.std()),
        "mag_rms": float(np.sqrt(np.mean(magnitude ** 2))),
        "mag_range": float(magnitude.max() - magnitude.min()),
        "dominant_bin": float(spectrum.argmax()),                 # times hz / n_samples gives Hz
    }
```

> **Fill in:** how your project synchronized devices, the window length and overlap, the normalization, and whether the model saw features, raw values or both.

## 4. Classic baselines

| Model | Input | Strengths | Weaknesses |
|---|---|---|---|
| Random forest | Features | Strong, fast, little tuning, feature importances | Only as good as the features |
| SVM | Features | Good on small datasets | Scaling and kernel tuning needed |
| 1D CNN | Raw windows | Learns local motion patterns, fast inference | Needs more data |
| LSTM or CNN-LSTM | Raw windows | Captures temporal order | Slower to train, more tuning |

Why it matters for your project: examiners and interviewers will ask whether the LLM beat a simple model. If you compared against one, know the numbers; if you did not, say so and explain how you would.

## 5. How LLMs are applied to sensor data
Typical approaches (identify which one, or which mix, your project used):

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| Features as text | Summarize each window as statistics in a prompt; the model answers with a label | Compact prompts; works with any text model | Feature design still matters |
| Raw values as text | Downsampled numbers written into the prompt | No feature engineering | Numbers use many tokens; models reason poorly over long number lists |
| Fine-tuning (often LoRA or QLoRA) | Train on prompt-to-label pairs so the model learns the label set and format | Much better than zero-shot on a narrow task; small adapters | Needs labeled data and GPU time |
| Multimodal fusion | A sensor encoder produces embeddings projected into the LLM's embedding space, similar to how vision-language models handle images | Uses signals directly, not as text | More complex to build and train |
| LLM over a classifier | A classic model labels windows; the LLM summarizes, explains or uses context | Accurate and cheap recognition plus natural language | Two systems to maintain |
| In-context demonstrations | Labeled examples placed in the prompt (few-shot) | No training | Limited by context length and cost |

```python
def window_to_prompt(features: dict[str, dict[str, float]], seconds: float) -> str:
    """features per device, for example {"watch": {...}, "phone": {...}, "glasses": {...}}."""
    lines = [f"Sensor summary for a {seconds:.2f} s window:"]
    for device, stats in features.items():
        lines.append(f"- {device}: " + ", ".join(f"{name}={value:.2f}" for name, value in stats.items()))
    lines.append("Answer with exactly one label: walking, running, sitting, standing, stairs.")
    return "\n".join(lines)

# One fine-tuning example; the exact record format depends on the training tool you use.
example = {"prompt": window_to_prompt(features, 2.56), "completion": "walking"}
```

### Why use LLMs at all? The honest trade-off
- **Potential benefits:** one model can take heterogeneous devices through a shared text interface; outputs can include explanations or descriptions, not just labels; few-shot adaptation to new activities; context (time of day, previous activities) can be added as text.
- **Costs:** numbers tokenize poorly; long windows are expensive or exceed the context; inference is slower and heavier than a small CNN; on standard benchmarks, specialized models are often as accurate or better; on-device deployment is harder.
- Research in recent years has explored LLMs on sensor data with mixed results against specialized models, so a strong answer explains your goal (for example, natural-language "demonstrations" of activities) and shows a baseline comparison.

> **Fill in:** which approach your project used, the prompt or input format, why your team chose LLMs, and what "demonstrations" means in your title (visual demonstrations of recognized activities, generated descriptions, few-shot examples in prompts, or something else).

## 6. Evaluation without leakage
- **Split by subject, not by window.** With overlapping windows, a random split puts near-identical windows in train and test, and the model also gets to see every person during training. Scores look excellent and collapse on new users.
- **Leave-one-subject-out (LOSO):** train on all subjects but one, test on the held-out subject, repeat for everyone, report the mean and spread.
- **Fit normalization inside each training fold.**
- **Metrics:** macro F1 (classes count equally), per-class F1, a confusion matrix, accuracy as a secondary number. For LLM outputs, also report the parse rate (the share of answers that were a valid label) and latency per window.
- **Compare every model under the same splits.**

```python
from sklearn.metrics import f1_score
from sklearn.model_selection import LeaveOneGroupOut

scores = []
for train_idx, test_idx in LeaveOneGroupOut().split(X, y, groups=subjects):
    model = make_model()                                   # illustrative factory: baseline or LLM wrapper
    model.fit(X[train_idx], y[train_idx])
    scores.append(f1_score(y[test_idx], model.predict(X[test_idx]), average="macro"))
print(f"LOSO macro F1: {np.mean(scores):.3f} +/- {np.std(scores):.3f}")
```

> **Fill in:** how your project split the data, the metrics you reported, the results per class if you have them, and the activities most often confused.

## 7. Typical web architecture

> A typical setup for a .NET plus Python ML web app, to adapt to what you actually built.

```text
Browser
  | upload recordings, view timelines and demonstrations
  v
.NET web app (ASP.NET Core MVC or Blazor)
  - users, recordings, results (database via Entity Framework or similar)
  - typed HttpClient -> POST /predict  (JSON)
  v
Python model service (for example FastAPI or Flask)
  - sync, windowing, normalization
  - fine-tuned LLM loaded once at startup (GPU if available)
```

```csharp
using System.Net.Http.Json;

public sealed record PredictRequest(string RecordingId, IReadOnlyList<double[][]> Windows);
public sealed record PredictResponse(IReadOnlyList<string> Activities);

// C# 12 primary constructor; registered as a typed client below.
public sealed class ActivityModelClient(HttpClient http)
{
    public async Task<PredictResponse> PredictAsync(PredictRequest request, CancellationToken ct = default)
    {
        using var response = await http.PostAsJsonAsync("/predict", request, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PredictResponse>(cancellationToken: ct)
               ?? throw new InvalidOperationException("Empty response from the model service");
    }
}

// Program.cs
builder.Services.AddHttpClient<ActivityModelClient>(client =>
{
    client.BaseAddress = new Uri("http://model-service:8000");
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()
model = load_model()  # illustrative: load weights once at startup, never per request

class PredictRequest(BaseModel):
    recording_id: str = Field(alias="recordingId")   # .NET sends camelCase JSON by default
    windows: list[list[list[float]]]                 # windows x samples x channels

class PredictResponse(BaseModel):
    activities: list[str]

@app.post("/predict", response_model=PredictResponse)
def predict(req: PredictRequest) -> PredictResponse:
    return PredictResponse(activities=[model.predict(w) for w in req.windows])
```

Design points: load the model once; batch windows per request; set timeouts on the .NET side; for long recordings, process asynchronously (a job queue and a status endpoint) instead of holding a request open; keep the contract versioned so either side can change independently.

> **Fill in:** the real .NET stack (MVC, Blazor, Razor Pages), the Python framework, how the two communicated, where the model ran, and where data and results were stored.

## Common pitfalls
- Random window-level splits that leak subjects and overlapping windows, inflating results.
- Normalizing with statistics from the whole dataset.
- Reporting accuracy alone on imbalanced activity classes.
- No classic baseline next to the LLM.
- Ignoring unparseable LLM answers instead of counting them as errors.
- Unsynchronized devices, so windows from different devices describe different moments.
- Loading the model per request in the Python service.
- Claiming results or design choices that were a teammate's work.

## How to talk about this in an interview
- **It is academic work; present it that way.** It shows you can build a web app that integrates a Python model service, work with data, and evaluate a model honestly.
- **Be precise about your role.** If it was a team project (common for FYPs), say "we" for the project and "I" for your parts; if you worked alone, change the "we" in the one-minute template to "I".
- **Know your numbers or say you do not.** Never improvise a metric; "I don't remember the exact figure, but the LLM was [better or worse] than the baseline on [metric]" is acceptable only if true.
- **Connect it to the job:** for full-stack roles, emphasize the .NET to Python integration and data handling; for AI-curious teams, emphasize evaluation without leakage and the LLM trade-offs.

> **Fill in:** two things you would do differently with more time, and what you learned that you now use at work.

## Interview questions

**Q1. Explain your FYP in one minute.**
Use the template in section 1: the problem (recognizing activities from smartwatch, phone and glasses data), the system (.NET web app plus a Python service with a fine-tuned LLM), the data and evaluation, the result against a baseline, and your role.
> **Fill in:** your rehearsed version.

**Q2. Why LLMs for activity recognition instead of a CNN or a random forest?**
Give your team's actual goal, then the honest trade-off: LLMs offer a shared text interface across devices, natural-language outputs and few-shot flexibility; specialized models are cheaper, faster and often as accurate. Then the evidence: how your LLM compared with a baseline.
> **Fill in:** your team's reason and the comparison.

**Q3. Which metrics did you use, and why?**
Strong answer: macro F1 because activity classes are imbalanced, per-class F1 and a confusion matrix to see which activities get confused, accuracy as a secondary figure, plus parse rate and latency for the LLM. State what you actually reported.
> **Fill in:** your metrics and results.

**Q4. How did you avoid data leakage?**
Subject-wise splits or leave-one-subject-out, normalization fitted on training folds, and no overlapping windows shared between train and test. If your project used random splits, admit it and explain the fix.

**Q5. How did you synchronize the smartwatch, phone and glasses?**
A shared clock, a sync event or cross-correlation for alignment, resampling to one rate, and a policy for gaps and missing devices. Describe what you did.

**Q6. How did you choose the window size and overlap?**
Activity duration and responsiveness: a few seconds captures a full gait cycle; 50% overlap is a common default; ideally tuned on validation data. Mention the UCI HAR setup (2.56 s, 50% overlap at 50 Hz) as a reference point.

**Q7. Which LLM did you fine-tune, and how?**
Name the model and the method (LoRA, QLoRA, full fine-tuning, a hosted fine-tuning service), the training data size, and the hardware. Explain why parameter-efficient fine-tuning made it feasible, if you used it.
> **Fill in:** model, method, data size, hardware.

**Q8. How did you represent sensor data for the LLM?**
Features as text, raw values as text, or embeddings through a sensor encoder. Explain the token cost and what you did to keep prompts short.

**Q9. What baseline did you compare against, and how did the LLM do?**
Name the baseline (for example a random forest on features or a 1D CNN) and the comparison under the same splits. If the LLM lost on accuracy, explain what it offered instead.

**Q10. What does "demonstrations" mean in your title?**
Answer precisely: visual demonstrations of recognized activities, generated descriptions, in-context examples in prompts, or something else.
> **Fill in:** the exact meaning.

**Q11. How does the web app communicate with the model?**
A typed HTTP client in .NET posting JSON to a Python service that loads the model once; timeouts; asynchronous processing for long recordings; a stable request and response contract.

**Q12. What was your role in the team?**
Your modules, decisions and code, with one concrete challenge you solved; the rest is "we". If you worked alone, say so and name the part that was hardest.
> **Fill in:** your role.

**Q13. What are the limitations, and what would you do next?**
Typical limitations: few subjects, lab conditions instead of real life, device placement variability, LLM latency and cost. Next steps: more diverse data, on-device models, real-time streaming, a stronger baseline comparison.

**Q14. What privacy concerns does sensor data raise?**
Motion data can reveal health conditions, habits and even identity; glasses with cameras capture bystanders. Collect with informed consent, minimize and anonymize data, secure storage and transport, and prefer on-device processing where possible.

## Further reading
- scikit-learn, cross-validation (group-based splitters): https://scikit-learn.org/stable/modules/cross_validation.html
- Hugging Face PEFT documentation: https://huggingface.co/docs/peft
- ASP.NET Core, making HTTP requests with IHttpClientFactory: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests
