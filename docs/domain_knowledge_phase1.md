# Phase 1 — Industrial Domain Knowledge

> Project: Industrial Predictive Maintenance & Anomaly Detection  
> Scope: Applied domain knowledge for an AI/ML engineer. This is not a mechanical-engineering reference.

## 1. Maintenance strategies

- **Reactive (run-to-failure):** repair equipment after it fails. Simple, but can cause unplanned downtime and secondary damage.
- **Preventive (time/usage-based):** maintain or replace components at predetermined intervals. Reduces some unexpected failures, but may service healthy components too early and can miss failures between intervals.
- **Predictive (condition-based/data-driven):** use equipment-condition observations and analytical models to support maintenance before failure. It can improve timing, but depends on reliable data, useful models, and sensible operational thresholds.

Predictive maintenance does not guarantee that failures will be prevented. It supports better-informed maintenance decisions.

## 2. Four related but distinct ML problems

| Problem | Main question | Typical formulation |
|---|---|---|
| Fault detection / diagnosis | Is a known fault present now, and which one? | Binary or multiclass classification |
| Anomaly detection | Does current behaviour differ from expected behaviour? | Unsupervised, semi-supervised, or supervised scoring |
| Failure prediction | Will a defined failure occur within a future horizon? | Binary classification, survival modelling |
| Remaining Useful Life (RUL) | How much operating life remains before a defined endpoint? | Regression or survival modelling |

An anomaly is not automatically a failure. A change in load, speed, environment, or sensor behaviour can produce unusual readings without equipment damage. An anomaly score is not automatically a calibrated failure probability.

## 3. Sensors and telemetry

Industrial sensors measure physical quantities; they do not directly measure “health.” Common examples include:

- **Vibration / accelerometers:** mechanical oscillation; useful for condition monitoring and signal analysis.
- **Temperature:** thermal condition; affected by load, ambient conditions, cooling, and friction.
- **Current:** electrical behaviour and load-related changes.
- **Speed:** rotational operating state; useful context for interpreting other signals.
- **Pressure:** fluid or gas system behaviour in pumps, compressors, and process equipment.

Interpret sensor values in operating context. For example, higher vibration at higher rotational speed is not necessarily evidence of deterioration. Include operating variables, compare like-for-like operating regimes, or use condition-aware baselines where justified.

### Signal representations

1. **Raw waveform:** dense, sampled signal that retains detailed time/frequency information.
2. **Window-level features:** summaries such as mean, standard deviation, RMS, peak-to-peak amplitude, kurtosis, and spectral features.
3. **Aggregated telemetry:** time-indexed summaries (for example, one record per minute).

These are alternative representations or processing stages, not a mandatory sequence. Choose based on what the dataset actually contains and what the task requires.

### Sampling frequency

For sampling interval \(\Delta t\), sampling frequency is \(f_s=1/\Delta t\). The Nyquist condition requires sampling above twice the highest frequency of interest (with practical margin and appropriate anti-alias filtering). Sampling that is too slow can lose rapid events or cause aliasing. Different sensor types can require different sampling rates.

## 4. Time-series concepts relevant to modelling

- **Univariate:** one measured variable over time.
- **Multivariate:** multiple aligned variables over time; enables contextual relationships between sensors.
- **Trend:** gradual movement in a signal, potentially reflecting degradation or changing workload.
- **Seasonality / cycles:** recurring patterns, such as shift- or duty-cycle-related behaviour.
- **Autocorrelation:** dependence between a signal and its lagged values; nearby samples are often not independent.
- **Stationarity:** stable statistical properties over time (in the relevant weak/strict sense). Not every ML model requires stationary inputs, but changing distributions and trends matter for validation and feature design.

Data-quality issues include missing telemetry, irregular timestamps, noise, spikes, sensor drift, stuck sensors, and misaligned sensor clocks. A sensor fault is not automatically an equipment failure. Missingness can itself be informative, so investigate why data is absent before imputing it.

## 5. Constructing a supervised prediction problem

First define the operational event precisely. A failure label should come from trustworthy event records (for example, validated work orders or confirmed failure logs), not simply from an unusual sensor value. Specify what counts as failure, which component/mode is in scope, the event timestamp, and how repairs or returns to service affect the timeline.

For a prediction timestamp \(t\):

- **Observation window \(W\):** historical sensor data available up to and including \(t\).
- **Prediction horizon \(H\):** future interval over which we ask whether a failure occurs.
- **Feature vector:** \(z_t=\phi(x_{t-W+1:t})\), computed only from information available by \(t\).
- **Binary target:** \(y_t=1\) if a defined failure occurs in \((t,t+H]\); otherwise \(y_t=0\), provided the full horizon is observed and the label is trustworthy.

The interval is written open at \(t\) so a failure that has already happened is not treated as a future event. The horizon and operational definition must be chosen with the maintenance team/use case in mind; they are not universal constants.

### Example

At 10:00, use the previous 60 minutes of sensor data to predict whether a defined failure will occur during the next 24 hours. If a confirmed failure occurs at 18:00, that 10:00 example is positive. If no failure occurs during the fully observed next 24 hours, it can be negative. If the dataset ends at 16:00, the example is **censored/unknown**, not safely negative, because the complete horizon was not observed.

## 6. Sliding windows and leakage

With window length \(W\) and stride \(S\), successive windows may overlap. Overlap increases the number of examples but does not create independent observations. Adjacent windows can be highly similar and may share most of their raw data.

Avoid these errors:

- Randomly splitting highly overlapping windows across train and test sets.
- Using readings after prediction time \(t\) to compute features.
- Letting a prediction horizon cross a dataset boundary and marking an incomplete horizon negative.
- Using maintenance actions, post-failure status fields, or other information only available after the event as predictors.
- Allowing windows from the same machine or failure episode to leak across splits when the evaluation goal is unseen-machine or future-event generalization.

Use chronological splits for future-in-time evaluation. Use group-based splits by machine/unit when testing generalization to unseen equipment. The appropriate strategy depends on the deployment claim; report exactly what the test split represents.

## 7. Class imbalance and operational evaluation

Failures are often rare relative to normal operation. A model that predicts “no failure” most of the time can achieve high accuracy while missing the events that matter.

Useful metrics include:

- **Precision:** of predicted failures, how many were actual failures?
- **Recall:** of actual failures, how many did we detect?
- **F1:** harmonic mean of precision and recall.
- **PR-AUC:** informative for rare positive classes; its baseline depends on positive prevalence.
- **ROC-AUC:** measures ranking across thresholds, but can look reassuring under severe imbalance.
- **Specificity / false-positive rate:** how well normal cases are left unflagged.

Choose thresholds using operational costs: missed failures, unnecessary inspections, downtime caused by inspections, and maintenance capacity. Where labels and timestamps permit, also evaluate event-level detection, false alarms per machine-time, and warning lead time—not only row-level metrics.

## 8. Key modelling principles

1. Sensor readings are evidence about physical behaviour, not direct health labels.
2. Separate equipment anomalies, sensor faults, known faults, future failures, and RUL.
3. Include operating context to avoid confusing normal workload changes with degradation.
4. Respect time: features must use only past/current information, and validation must reflect deployment.
5. Treat incomplete future horizons as unknown/censored, not automatically negative.
6. Evaluate maintenance usefulness, not accuracy alone.
7. Never claim a capability, metric, or deployment component until it has been implemented and tested.

## 9. Phase 1 completion checklist

- [ ] Explain reactive, preventive, and predictive maintenance.
- [ ] Distinguish fault detection, anomaly detection, failure prediction, and RUL.
- [ ] Explain what common sensors measure and why operating context matters.
- [ ] Explain raw waveforms, telemetry, sampling frequency, and basic time-series properties.
- [ ] Define a failure event, observation window, prediction horizon, and target label.
- [ ] Explain censoring, overlapping windows, temporal leakage, and class imbalance.
- [ ] Explain why operational metrics and thresholds matter.

**Phase 1 exit criterion:** We can describe the industrial problem, identify what data and labels would be needed, and formulate a leakage-aware ML task—without yet committing to a dataset or algorithm.
