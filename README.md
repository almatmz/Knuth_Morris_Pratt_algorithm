# KMP String Matching

This project implements the Knuth–Morris–Pratt (KMP) string matching algorithm in Java (Maven), instruments the algorithm to count core operations, and validates the theoretical complexity with empirical results on datasets.

The current experiments search for the pattern `goal` (lowercase, case-sensitive) in three texts of different lengths: small, medium, and large.

---

## Quick Start

- Build and test
```bash
mvn clean verify
```

- Run the driver (auto-discovers and processes every JSON file in `input/`)
```bash
mvn -q exec:java -Dexec.mainClass="almat.Main"
```

- Outputs are written to `output/`:
    - One JSON per dataset: `output_<dataset>.json`
    - A consolidated `summary.csv` with milliseconds (ms)


---

## Project Structure

```
input/              # dataset JSON files (dataset, pattern, text)
output/             # generated JSON results + summary.csv
src/
  main/java/almat/
    KMPMatcher.java       # KMP implementation + instrumentation hooks
    OperationCounter.java # counts: charComparisons, fallbackSteps, lpsComputations, matchFallbacks
    DatasetProcessor.java # IO + run + pack results
    JsonIO.java           # Jackson helpers
    Main.java             # batch driver, scans input/, writes output/
  test/java/almat/
    KMPMatcherTest.java   # correctness + edge cases
    PerformanceTest.java  # empirical scaling via operation counts (stable)
README.md       
pom.xml             # Maven config (JUnit, JaCoCo, Checkstyle, SpotBugs)
```

---

## Algorithm Overview (KMP)

Given a pattern `P` of length `m` and text `T` of length `n`, KMP:

1. Preprocesses `P` to build an LPS array (Longest Proper Prefix which is also Suffix) in O(m).
2. Scans `T` in O(n), using LPS to skip redundant comparisons.
3. Finds all occurrences (including overlapping matches).

Complexity:
- Time: O(n + m)
- Space: O(m)

For the pattern `goal`, all characters are distinct. Its LPS array values are `[0,0,0,0]` and LPS construction performs `m-1 = 3` steps with our instrumentation.

---

## Instrumented Operations

We record the following counters per dataset:

- charComparisons: number of direct `text[i] == pattern[j]` comparisons
- fallbackSteps: mismatch-driven fallbacks during search (when `j = lps[j-1]` after a mismatch)
- matchFallbacks: post-match resets to allow overlaps (when `j = lps[j-1]` after a full match)
- lpsComputations: steps taken while building the LPS array (amortized O(m))
- elapsedMillis: end-to-end search time for a single run (milliseconds)

Note: The CSV and JSON include `elapsedMillis`. The JSON samples below omit `matchFallbacks` only because the pattern `goal` causes zero overlap resets (LPS ends in 0); the counter exists in code and will be non-zero on overlap-heavy patterns.

---

## Datasets

All experiments use pattern `goal`.

- football_small: a short, single-occurrence sentence
- football_medium: a paragraph with multiple mentions
- football_large: multi-paragraph text with many mentions

---

## Results 

### Summary (CSV-style)
| dataset         | textLength | patternLength | matches | charComparisons | fallbackSteps | lpsComputations | elapsedMillis |
|----------------|------------|---------------|---------|-----------------|---------------|-----------------|---------------|
| football_large | 611        | 4             | 9       | 617             | 6             | 3               | 1.415         |
| football_medium| 289        | 4             | 4       | 291             | 2             | 3               | 0.040         |
| football_small | 62         | 4             | 1       | 62              | 0             | 3               | 0.010         |

Derived indicators:
- Comparisons per character (charComparisons / textLength):
    - large: 1.010
    - medium: 1.007
    - small: 1.000
- Time per character (μs/char):
    - large: 2.316
    - medium: 0.138
    - small: 0.161
- Match density (matches per 1K chars):
    - large: 14.737
    - medium: 13.840
    - small: 16.129

These indicators match KMP’s linear scanning: comparisons ≈ n, minor constant factors, and negligible dependence on the number of matches when the pattern has little self-overlap.

### Per-dataset JSON results 

```json
{
  "dataset" : "football_large",
  "pattern" : "goal",
  "textLength" : 611,
  "matches" : [ 64, 111, 155, 247, 307, 384, 487, 563, 594 ],
  "charComparisons" : 617,
  "fallbackSteps" : 6,
  "lpsComputations" : 3,
  "elapsedMillis" : 1.4148
}
```

```json
{
  "dataset" : "football_medium",
  "pattern" : "goal",
  "textLength" : 289,
  "matches" : [ 78, 132, 171, 284 ],
  "charComparisons" : 291,
  "fallbackSteps" : 2,
  "lpsComputations" : 3,
  "elapsedMillis" : 0.0397
}
```

```json
{
  "dataset" : "football_small",
  "pattern" : "goal",
  "textLength" : 62,
  "matches" : [ 17 ],
  "charComparisons" : 62,
  "fallbackSteps" : 0,
  "lpsComputations" : 3,
  "elapsedMillis" : 0.0099
}
```

---

## Empirical Validation: Theory vs Practice

Theoretical expectations:
- LPS Build: O(m) — for `goal` (m=4), we observe `lpsComputations = 3` (exactly m−1 iterations).
- Search: O(n) — comparisons should be tightly bounded by n, with occasional fallbacks when mismatches occur.

Observed behavior:
- charComparisons ≈ textLength for all datasets (ratios ~1.00–1.01), confirming near-ideal linear scanning.
- fallbackSteps are small (0–6) due to the pattern’s lack of self-overlap; KMP rarely needs to retreat along LPS in such cases.
- elapsedMillis scales with input size broadly, though microbenchmarks can vary due to JIT, GC, and OS scheduling. We treat time as a sanity check; comparisons are the primary empirical proxy for algorithmic work.
- matchFallbacks (post-match resets enabling overlaps) remain 0 for `goal` because the LPS of the final position is 0. On overlap-heavy patterns (e.g., `aa…a`), this counter increases and still exhibits linear behavior.

Conclusion:
- The empirical data align with KMP’s O(n + m) complexity.
- Operation counts (especially charComparisons) provide a stable, noise-free validation of linear-time behavior across text sizes.

---


## Testing

- Correctness tests cover:
    - Exact match, no match, multiple and overlapping matches
    - Start/middle/end placements
    - Pattern longer than text, empty text/pattern, null inputs
    - Unicode content
    - Canonical LPS example (“ababaca”)

- Performance test focuses on scaling via operation counts (charComparisons), not wall-clock time—making it stable across machines and JDKs.

Run all tests:
```bash
mvn test
```

---



## Conclusion

- We implemented KMP correctly and instrumented it to count character comparisons, LPS steps, and fallbacks.
- On football_small/medium/large with pattern “goal”, the empirical operation counts match the theory:
    - charComparisons ≈ textLength (1.00–1.01×), confirming O(n) search after O(m) preprocessing (lpsComputations = 3 for m = 4).
    - fallbackSteps are very small (0–6) because “goal” has no self-overlap; KMP rarely needs to retreat along the LPS.
    - elapsedMillis is low and broadly proportional to input size, with normal variability from JIT/GC/OS scheduling.
- Therefore, the experiments validate KMP’s O(n + m) time and O(m) space in practice.
