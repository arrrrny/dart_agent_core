Understanding non-deterministic agent behavior requires metrics that go beyond simple pass/fail rates. The `dart_agent_core` evaluation framework implements two complementary statistical estimators — **pass@k** and **pass^k** — that quantify different aspects of agent reliability across multiple trials. This documentation explains their mathematical foundations, implementation details, and practical usage within the framework.

## Mathematical Foundations

Both metrics address the fundamental challenge of measuring reliability when agent behavior is non-deterministic. Given `n` total trials with `c` successes (where `c ≤ n`), these metrics answer distinct questions about performance across `k` attempts.

### pass@k: The Codex Unbiased Estimator

**pass@k** answers: *"What is the probability that at least one of `k` independent attempts will succeed?"*

The implementation uses the unbiased combinatorial formula from the Codex paper:

```
pass@k = 1 − C(n − c, k) / C(n, k)
```

Where `C(a, b)` represents the binomial coefficient "a choose b". This formula avoids the bias that naive estimators introduce when `n` is small relative to `k`.

**Key properties:**
- Returns values in `[0.0, 1.0]`
- When `k = 1`, reduces to the simple pass rate `c/n`
- As `k` increases toward `n`, approaches `1.0` if any successes exist
- Handles edge cases: returns `0.0` when `k > n` or `k ≤ 0`

Sources: [pass_at_k.dart](lib/src/eval/metrics/pass_at_k.dart#L1-L25)

### pass^k: The Empirical Estimator

**pass^k** answers: *"What is the probability that all `k` independent attempts will succeed?"*

The implementation uses the empirical estimator:

```
pass^k = (c/n)^k
```

This measures **sustained reliability** — the likelihood of achieving consistent success across multiple attempts. Unlike pass@k, this metric penalizes inconsistency heavily.

**Key properties:**
- Returns values in `[0.0, 1.0]`
- When `k = 1`, also reduces to `c/n`
- Decays exponentially as inconsistency increases
- More sensitive to failure patterns than pass@k

Sources: [pass_caret_k.dart](lib/src/eval/metrics/pass_caret_k.dart#L1-L25)

## Implementation Architecture

The framework integrates these metrics through a layered architecture that separates computation from reporting.

### Core Metric Functions

The metric functions are pure, stateless transformations that operate on boolean lists:

```dart
// pass@k computation
double passAtK(List<bool> trialPasses, int k) {
  final n = trialPasses.length;
  if (k <= 0 || n == 0) return 0.0;
  if (k > n) return 0.0;
  final c = trialPasses.where((p) => p).length;
  if (n - c < k) return 1.0;
  
  // log-domain product to avoid overflow on big n
  var prob = 1.0;
  for (var i = 0; i < k; i++) {
    prob *= (n - c - i) / (n - i);
  }
  return 1.0 - prob;
}

// pass^k computation  
double passCaretK(List<bool> trialPasses, int k) {
  final n = trialPasses.length;
  if (k <= 0 || n == 0) return 0.0;
  final c = trialPasses.where((p) => p).length;
  if (c == 0) return 0.0;
  return _pow(c / n, k);
}
```

Sources: [pass_at_k.dart](lib/src/eval/metrics/pass_at_k.dart#L10-L25), [pass_caret_k.dart](lib/src/eval/metrics/pass_caret_k.dart#L10-L25)

### Report Integration

The `EvalRunReport` class provides task-level aggregation methods that compute both metrics across configurable `k` values:

```dart
// Compute pass@k for k = 1, 3, 5 across all tasks
final passAtK = report.passAtKByTask(ks: [1, 3, 5]);

// Compute pass^k for the same k values
final passCk = report.passCaretKByTask(ks: [1, 3, 5]);

// Result structure: Map<String, Map<int, double>>
// {task_id: {k: metric_value}}
```

Sources: [eval_run_report.dart](lib/src/eval/core/eval_run_report.dart#L35-L48)

### Markdown Report Generation

The reporting system automatically generates comparative tables showing both metrics side-by-side:

```dart
// In report_generator.dart
final headers = [
  'Task',
  ...ksToReport.expand((k) => ['pass@$k', 'pass^$k']),
];
```

This produces tables like:

| Task | pass@1 | pass^1 | pass@3 | pass^3 |
|---|---|---|---|---|
| `task_simple_addition` | 100.0% | 100.0% | 100.0% | 100.0% |
| `task_multi_step` | 75.0% | 75.0% | 98.4% | 42.2% |

Sources: [report_generator.dart](lib/src/eval/reporting/report_generator.dart#L60-L75)

## Practical Usage Patterns

### Choosing Between Metrics

The choice between pass@k and pass^k depends on your evaluation goals:

| Metric | Use Case | Interpretation |
|---|---|---|
| **pass@1** | Baseline capability | Simple pass rate for single attempts |
| **pass@k (k>1)** | Best-of-k reliability | Probability of success with retries |
| **pass^1** | Baseline consistency | Same as pass@1 |
| **pass^k (k>1)** | Sustained reliability | Probability of consistent success |

**Anthropic's recommendation:** Target **pass@1 close to 100%** for baseline capability, and **maximize k for pass^k** to measure sustained reliability.

Sources: [eval-guide.md](doc/eval-guide.md#L887-L889)

### Configuration in Task Design

The `trialsPerRun` field in `EvalTask` directly impacts metric meaningfulness:

```dart
class _SimpleAdditionTask implements EvalTask {
  @override
  int get trialsPerRun => 2;  // Minimum for meaningful pass^k
  
  // With trialsPerRun=1, pass^k degenerates to pass@1
  // and tells you nothing about consistency
}
```

**Guidelines:**
- **Capability suites:** `trialsPerRun = 2-5` to measure non-determinism
- **Regression suites:** `trialsPerRun = 1` (consistency assumed)
- **Mixed suites:** Align with primary evaluation goal

Sources: [tasks.dart](example/eval_demo/calculator/tasks.dart#L45-L46), [eval-guide.md](doc/eval-guide.md#L100)

### Suite Kind Influence

The `SuiteKind` enum determines how metrics are interpreted at the task level:

| Suite Kind | Task Pass Criteria | Metric Focus |
|---|---|---|
| `capability` | At least one trial passes (pass@N) | Exploration, discovery |
| `regression` | All trials must pass (pass^N) | Consistency, reliability |
| `mixed` | Same as regression | Conservative approach |

Sources: [eval_suite.dart](lib/src/eval/core/eval_suite.dart#L5-L15)

## Interpretation and Decision Framework

### Reading Metric Results

Consider these patterns when interpreting results:

**Pattern 1: High pass@k, Low pass^k**
- Agent succeeds with retries but lacks consistency
- Example: pass@3 = 98%, pass^3 = 65%
- **Action:** Focus on reducing non-determinism (temperature, prompting)

**Pattern 2: Low pass@k, High pass^k**  
- Rarely succeeds, but when it does, it's consistent
- Example: pass@3 = 45%, pass^3 = 40%
- **Action:** Improve base capability (model selection, tool design)

**Pattern 3: Both metrics high**
- Reliable and consistent
- Example: pass@3 = 99%, pass^3 = 95%
- **Action:** Graduate task to regression suite

**Pattern 4: Both metrics low**
- Fundamental capability gap
- Example: pass@3 = 30%, pass^3 = 15%
- **Action:** Redesign task or agent architecture

### Saturation Analysis

The framework includes saturation analysis that uses pass@1 thresholds:

```dart
class SaturationThresholds {
  final double matureTaskPassRate = 0.95;  // pass@1 ≥ 95%
  final double brokenTaskPassRate = 0.0;   // pass@1 ≤ 0%
  final double saturatedSuiteRatio = 0.9;  // 90% tasks mature
}
```

When a capability suite reaches saturation (≥90% of tasks have pass@1 ≥ 95%), the framework suggests graduating stable tasks to regression suites.

Sources: [saturation_status.dart](lib/src/eval/suite_health/saturation_status.dart#L1-L15)

## Comparative Analysis

### Metric Sensitivity Comparison

| Scenario | n | c | k | pass@k | pass^k | Interpretation |
|---|---|---|---|---|---|---|
| Perfect consistency | 5 | 5 | 3 | 100% | 100% | Ideal state |
| High consistency | 5 | 4 | 3 | 100% | 51.2% | Good with occasional failures |
| Moderate consistency | 5 | 3 | 3 | 98.3% | 21.6% | Needs improvement |
| Low consistency | 5 | 2 | 3 | 90% | 6.4% | Significant issues |
| Minimal capability | 5 | 1 | 3 | 60% | 0.8% | Fundamental problems |

### Computational Complexity

Both metrics run in **O(k)** time with **O(1)** space, making them efficient for large-scale evaluations:

- **pass@k:** Single loop with multiplication
- **pass^k:** Simple exponentiation via multiplication

Sources: [pass_at_k.dart](lib/src/eval/metrics/pass_at_k.dart#L18-L22), [pass_caret_k.dart](lib/src/eval/metrics/pass_caret_k.dart#L18-L22)

## Integration with Evaluation Workflow

### Recording and Replay Considerations

When using recording/replay infrastructure, each trial must have a unique cache salt to preserve non-determinism:

```dart
// In Trial class
String get cacheSalt => '$taskId#$trialIndex';

// Without unique salts, trials collapse onto one cache entry
// and metrics no longer measure actual model non-determinism
```

Sources: [trial.dart](lib/src/eval/core/trial.dart#L85-L92)

### Cross-Run Analysis

The `SuiteHealthAnalyzer` uses pass@1 trends across runs to identify:
- **Graduation candidates:** Tasks with sustained high pass@1
- **Broken task candidates:** Tasks with consistently low pass@1
- **Saturation signals:** Suites where most tasks are mature

Sources: [saturation_status.dart](lib/src/eval/suite_health/saturation_status.dart#L50-L65)

## Next Steps

After understanding pass@k and pass^k metrics, explore these related topics:

1. **[Defining Tasks, Suites, and Graders](17-defining-tasks-suites-and-graders)** — How to design tasks that yield meaningful metric results
2. **[Trials, Outcomes, and Transcripts](18-trials-outcomes-and-transcripts)** — The raw data that feeds these metrics  
3. **[Suite Health Analysis and Graduation](22-suite-health-analysis-and-graduation)** — Using metrics for cross-run decision making
4. **[Recording, Replay, and Rate Limiting](20-recording-replay-and-rate-limiting)** — Preserving metric validity in CI environments

The combination of pass@k and pass^k provides a nuanced view of agent reliability that simple pass rates cannot capture. By understanding when to use each metric and how to interpret their relationship, you can make informed decisions about agent improvements and suite management.