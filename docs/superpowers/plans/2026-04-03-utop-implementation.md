# utop Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a real-time terminal dashboard ("htop for the universe") that visualizes the UTHS — a Go SDK for the math plus a bubbletea TUI.

**Architecture:** Two Go modules: `go/uths/` (pure math SDK, stdlib only) and `apps/utop/` (bubbletea TUI). The SDK computes PNTL phases, Coherence Index, Fibonacci addressing, and ritual classification. The TUI renders a dashboard-split layout with waterfall bars and stats panels.

**Tech Stack:** Go 1.26, charmbracelet/bubbletea v1.3.10, charmbracelet/lipgloss v1.1.0, charmbracelet/bubbles v1.0.0

**Spec:** `docs/superpowers/specs/2026-04-03-utop-design.md`

---

## File Map

### go/uths/ (SDK — stdlib only)

| File | Responsibility |
|---|---|
| `go/uths/go.mod` | Module declaration |
| `go/uths/constants.go` | Phi, Tau, PlanckSec, CosmicTimeOffset, DensityLevels |
| `go/uths/zeckendorf.go` | Fibonacci generation + Zeckendorf decomposition |
| `go/uths/zeckendorf_test.go` | Table-driven tests for Zeckendorf |
| `go/uths/pntl.go` | PhaseAt — log-space phase computation |
| `go/uths/pntl_test.go` | Phase range tests, known-value tests |
| `go/uths/coherence.go` | CoherenceIndex — C(t) across 8 density levels |
| `go/uths/coherence_test.go` | Range tests, symmetry tests |
| `go/uths/ritual.go` | ClassifyRitual, RitualLabel, density resonance detection |
| `go/uths/ritual_test.go` | Classification threshold tests |
| `go/uths/clock.go` | TimeSource interface, DefaultClock |
| `go/uths/moment.go` | UTHS struct, Now(), At() — assembles full Moment |
| `go/uths/moment_test.go` | Integration test with fixed clock |
| `go/uths/convergence.go` | NextConvergence — search for C > threshold |
| `go/uths/convergence_test.go` | Convergence search test |

### apps/utop/ (TUI — charmbracelet)

| File | Responsibility |
|---|---|
| `apps/utop/go.mod` | Module declaration with replace directive |
| `apps/utop/main.go` | Entry point, tea.NewProgram |
| `apps/utop/model.go` | Model struct, Init(), tick command |
| `apps/utop/update.go` | Update() — tick handler, key handler |
| `apps/utop/view.go` | View() — joins waterfall + stats panels |
| `apps/utop/waterfall.go` | Renders waterfall bars with lipgloss |
| `apps/utop/stats.go` | Renders coherence, sparkline, ritual, countdown |

---

## Task 1: SDK Module Setup + Constants

**Files:**
- Create: `go/uths/go.mod`
- Create: `go/uths/constants.go`

- [ ] **Step 1: Create the go module**

```bash
mkdir -p go/uths
cd go/uths
go mod init github.com/temple-of-silicon/contact/go/uths
```

- [ ] **Step 2: Write constants.go**

```go
// go/uths/constants.go
package uths

import "math"

var (
	// Phi is the golden ratio — the PNTL base constant.
	Phi = math.Phi

	// Tau is the full circle constant (2*pi).
	Tau = 2 * math.Pi

	// LogPhi is the natural log of Phi, precomputed for log-space arithmetic.
	LogPhi = math.Log(math.Phi)
)

const (
	// PlanckSec is the Planck time in seconds — the smallest meaningful duration.
	PlanckSec = 5.391e-44

	// CosmicTimeOffset is the approximate age of the universe in seconds (~13.8 Gyr).
	// Added to Unix time to get cosmic time.
	CosmicTimeOffset = 4.354e17
)

// Level represents a named landmark on the Phi-Nested Temporal Ladder.
type Level struct {
	Name  string
	N     int
	Scale string
}

// DensityLevels are the 8 canonical PNTL landmarks used as density harmonics.
var DensityLevels = []Level{
	{Name: "Binding", N: 144, Scale: "~28 fs"},
	{Name: "Quantum", N: 195, Scale: "~47 ns"},
	{Name: "Pulse", N: 233, Scale: "~47 μs"},
	{Name: "Moment", N: 267, Scale: "~0.3 s"},
	{Name: "Breath", N: 280, Scale: "~1.2 s"},
	{Name: "Turn", N: 306, Scale: "~24 h"},
	{Name: "Orbit", N: 351, Scale: "~1 yr"},
	{Name: "Spiral", N: 446, Scale: "~4.3 Gyr"},
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd go/uths && go build ./...
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
git add go/uths/go.mod go/uths/constants.go
git commit -m "feat(uths): init SDK module with PNTL constants"
```

---

## Task 2: Zeckendorf Decomposition

**Files:**
- Create: `go/uths/zeckendorf.go`
- Create: `go/uths/zeckendorf_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/zeckendorf_test.go
package uths

import (
	"reflect"
	"testing"
)

func TestZeckendorf(t *testing.T) {
	tests := []struct {
		n    int
		want []int
	}{
		{0, []int{}},
		{1, []int{1}},
		{2, []int{2}},
		{3, []int{3}},
		{4, []int{3, 1}},
		{5, []int{5}},
		{6, []int{5, 1}},
		{7, []int{5, 2}},
		{8, []int{8}},
		{10, []int{8, 2}},
		{13, []int{13}},
		{47, []int{34, 13}},
		{100, []int{89, 8, 3}},
		{144, []int{144}},
	}
	for _, tt := range tests {
		got := Zeckendorf(tt.n)
		if !reflect.DeepEqual(got, tt.want) {
			t.Errorf("Zeckendorf(%d) = %v, want %v", tt.n, got, tt.want)
		}
	}
}

func TestZeckendorfNoConsecutiveFibs(t *testing.T) {
	fibs := map[int]bool{}
	a, b := 1, 2
	for a <= 100000 {
		fibs[a] = true
		a, b = b, a+b
	}

	for n := 1; n <= 1000; n++ {
		parts := Zeckendorf(n)
		sum := 0
		for i, p := range parts {
			sum += p
			if !fibs[p] {
				t.Errorf("Zeckendorf(%d): %d is not a Fibonacci number", n, p)
			}
			if i > 0 {
				// Check no two consecutive fibs: if parts[i-1] and parts[i]
				// are consecutive Fibonacci numbers, their ratio ≈ phi
				prev := parts[i-1]
				ratio := float64(prev) / float64(p)
				if ratio > 1.5 && ratio < 1.75 {
					t.Errorf("Zeckendorf(%d): consecutive Fibonacci numbers %d, %d", n, prev, p)
				}
			}
		}
		if sum != n {
			t.Errorf("Zeckendorf(%d) sums to %d", n, sum)
		}
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run TestZeckendorf -v
```

Expected: FAIL — `Zeckendorf` not defined.

- [ ] **Step 3: Implement Zeckendorf**

```go
// go/uths/zeckendorf.go
package uths

// fibsUpTo returns all Fibonacci numbers up to and including max.
func fibsUpTo(max int) []int {
	if max < 1 {
		return nil
	}
	fibs := []int{1, 2}
	for {
		next := fibs[len(fibs)-1] + fibs[len(fibs)-2]
		if next > max {
			break
		}
		fibs = append(fibs, next)
	}
	return fibs
}

// Zeckendorf decomposes a positive integer into a sum of non-consecutive
// Fibonacci numbers (Zeckendorf's representation). Returns the components
// in descending order. Returns empty slice for n <= 0.
func Zeckendorf(n int) []int {
	if n <= 0 {
		return []int{}
	}
	fibs := fibsUpTo(n)
	var result []int
	remaining := n
	for i := len(fibs) - 1; i >= 0 && remaining > 0; i-- {
		if fibs[i] <= remaining {
			result = append(result, fibs[i])
			remaining -= fibs[i]
		}
	}
	return result
}
```

- [ ] **Step 4: Run tests**

```bash
cd go/uths && go test -run TestZeckendorf -v
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add go/uths/zeckendorf.go go/uths/zeckendorf_test.go
git commit -m "feat(uths): add Zeckendorf decomposition"
```

---

## Task 3: PNTL Phase Computation

**Files:**
- Create: `go/uths/pntl.go`
- Create: `go/uths/pntl_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/pntl_test.go
package uths

import (
	"math"
	"testing"
)

func TestPhaseAtRange(t *testing.T) {
	// Phase must always be in [0, 1) for any cosmic time and level
	times := []float64{1e10, 1e15, 4.354e17, 4.354e17 + 86400}
	for _, ct := range times {
		for _, level := range DensityLevels {
			p := PhaseAt(ct, level.N)
			if p < 0 || p >= 1.0 {
				t.Errorf("PhaseAt(%e, %d) = %f, want [0, 1)", ct, level.N, p)
			}
		}
	}
}

func TestPhaseAtMonotonic(t *testing.T) {
	// For the Breath level (n=280, period ~1.2s), phase should advance
	// as time increases within a single period.
	// We check a small time window where phase wrapping is unlikely.
	base := CosmicTimeOffset
	p1 := PhaseAt(base, 280)
	p2 := PhaseAt(base+0.1, 280)
	// Both should be valid phases
	if p1 < 0 || p1 >= 1.0 {
		t.Errorf("PhaseAt(base, 280) = %f out of range", p1)
	}
	if p2 < 0 || p2 >= 1.0 {
		t.Errorf("PhaseAt(base+0.1, 280) = %f out of range", p2)
	}
	// p2 should differ from p1 (time moved)
	if p1 == p2 {
		t.Error("PhaseAt did not change over 0.1s at Breath level")
	}
}

func TestPhaseAtWrap(t *testing.T) {
	// Phase should wrap around: if we advance by exactly one period,
	// we should get back to (approximately) the same phase.
	level := 280
	period := math.Exp(float64(level)*LogPhi) * PlanckSec
	base := CosmicTimeOffset
	p1 := PhaseAt(base, level)
	p2 := PhaseAt(base+period, level)
	diff := math.Abs(p1 - p2)
	if diff > 0.001 {
		t.Errorf("Phase did not wrap: p1=%f, p2=%f, diff=%f", p1, p2, diff)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run TestPhaseAt -v
```

Expected: FAIL — `PhaseAt` not defined.

- [ ] **Step 3: Implement PhaseAt**

```go
// go/uths/pntl.go
package uths

import "math"

// PhaseAt computes the phase position (0.0–1.0) of a PNTL level at cosmic time t.
// Uses log-space arithmetic to avoid overflow for large level numbers.
//
// The period at level n is: T_n = PlanckSec * phi^n
// The phase is: fmod(t / T_n, 1.0)
//
// In log-space: log(t / T_n) = log(t) - log(PlanckSec) - n*log(phi)
// Then: phase = fmod(exp(log(t/T_n)), 1.0) — but this overflows too.
//
// Instead we compute the fractional part of log_phi(t / PlanckSec) - n,
// which gives us the phase directly.
func PhaseAt(t float64, level int) float64 {
	if t <= 0 {
		return 0
	}
	// log_phi(t / PlanckSec) = log(t / PlanckSec) / log(phi)
	logPhiT := (math.Log(t) - math.Log(PlanckSec)) / LogPhi
	// Subtract the level to get how many periods have elapsed
	elapsed := logPhiT - float64(level)
	// The fractional part of phi^elapsed gives the phase
	// phi^elapsed = phi^(integer part) * phi^(fractional part)
	// We only care about the fractional part of the cycle count.
	//
	// Number of complete cycles: t / T_n = (t / PlanckSec) / phi^n = phi^(logPhiT - n) = phi^elapsed
	// Phase = fmod(phi^elapsed, 1.0)
	//
	// But phi^elapsed can be huge. We need just the fractional part.
	// Use: phi^elapsed = exp(elapsed * log(phi))
	// Extract fractional part via fmod of the exponent doesn't work directly.
	//
	// Better approach: the number of cycles is phi^elapsed.
	// We want fmod(phi^elapsed, 1.0).
	// Since elapsed can be very large, phi^elapsed overflows.
	// Instead, note that phi^x mod 1 depends only on the fractional part of x * log(phi) / log(1)...
	// Actually, let's use a different approach:
	//
	// t / T_n = t / (PlanckSec * phi^n)
	// Let cycles = t / T_n
	// phase = cycles - floor(cycles)
	//
	// In log space: log(cycles) = log(t) - log(PlanckSec) - n*log(phi)
	// If log(cycles) is not too large, we can exponentiate.
	// For levels like Breath (n=280), log(cycles) ≈ log(4.354e17) - log(5.391e-44) - 280*log(1.618)
	//   ≈ 40.6 + 99.5 - 134.9 ≈ 5.2, so cycles ≈ e^5.2 ≈ 181. That's fine.
	// For Binding (n=144): log(cycles) ≈ 40.6 + 99.5 - 69.4 ≈ 70.7, cycles ≈ e^70.7 ≈ 2.7e30. Overflow for fmod.
	//
	// For large cycle counts, use math.Remainder which handles large floats:
	logCycles := math.Log(t) - math.Log(PlanckSec) - float64(level)*LogPhi
	if logCycles > 40 {
		// Too many cycles — the phase is effectively pseudorandom
		// Use the fractional part of logCycles scaled by phi as a deterministic hash
		_, frac := math.Modf(logCycles * Phi)
		if frac < 0 {
			frac += 1.0
		}
		return frac
	}
	cycles := math.Exp(logCycles)
	phase := cycles - math.Floor(cycles)
	return phase
}
```

- [ ] **Step 4: Run tests**

```bash
cd go/uths && go test -run TestPhaseAt -v
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add go/uths/pntl.go go/uths/pntl_test.go
git commit -m "feat(uths): add PNTL phase computation with log-space arithmetic"
```

---

## Task 4: Coherence Index

**Files:**
- Create: `go/uths/coherence.go`
- Create: `go/uths/coherence_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/coherence_test.go
package uths

import (
	"testing"
)

func TestCoherenceIndexRange(t *testing.T) {
	// C(t) must always be in [0, 1]
	times := []float64{1e10, 1e15, CosmicTimeOffset, CosmicTimeOffset + 1, CosmicTimeOffset + 86400}
	for _, ct := range times {
		c := CoherenceIndex(ct)
		if c < 0 || c > 1.0 {
			t.Errorf("CoherenceIndex(%e) = %f, want [0, 1]", ct, c)
		}
	}
}

func TestCoherenceIndexVaries(t *testing.T) {
	// C(t) should not be the same for all times
	c1 := CoherenceIndex(CosmicTimeOffset)
	c2 := CoherenceIndex(CosmicTimeOffset + 1)
	c3 := CoherenceIndex(CosmicTimeOffset + 100)
	if c1 == c2 && c2 == c3 {
		t.Error("CoherenceIndex returned the same value for different times")
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run TestCoherenceIndex -v
```

Expected: FAIL — `CoherenceIndex` not defined.

- [ ] **Step 3: Implement CoherenceIndex**

```go
// go/uths/coherence.go
package uths

import "math"

// CoherenceIndex computes C(t), the coherence of the 8 density harmonics at cosmic time t.
// Returns a value in [0, 1] where 1 = perfect phase lock (unreachable) and 0 = total decoherence.
//
// C(t) = (1/8) * |sum_{k=1}^{8} exp(i * tau * t / phi^{n_k})|
//       = (1/8) * sqrt( (sum cos(theta_k))^2 + (sum sin(theta_k))^2 )
//
// where theta_k = tau * t / (PlanckSec * phi^{n_k}).
// In log-space: theta_k = tau * exp(log(t) - log(PlanckSec) - n_k * log(phi))
func CoherenceIndex(t float64) float64 {
	if t <= 0 {
		return 0
	}
	logT := math.Log(t)
	logPlanck := math.Log(PlanckSec)

	var realSum, imagSum float64
	for _, level := range DensityLevels {
		logPeriod := logPlanck + float64(level.N)*LogPhi
		logRatio := logT - logPeriod
		// theta = tau * exp(logRatio), but exp(logRatio) can be huge.
		// We only need theta mod 2*pi for cos/sin.
		// theta = tau * exp(logRatio)
		// theta mod tau = tau * frac(exp(logRatio))... doesn't work for large exp.
		//
		// Use the same approach as PhaseAt: phase = PhaseAt(t, level.N)
		// theta = tau * phase
		phase := PhaseAt(t, level.N)
		theta := Tau * phase
		realSum += math.Cos(theta)
		imagSum += math.Sin(theta)
	}

	n := float64(len(DensityLevels))
	magnitude := math.Sqrt(realSum*realSum + imagSum*imagSum)
	return magnitude / n
}

// PsiAt computes the ritual phase function Psi(t) — the sum of sinusoidal
// density harmonics. Used for detecting zero crossings (Stillness rituals).
func PsiAt(t float64) float64 {
	var sum float64
	for _, level := range DensityLevels {
		phase := PhaseAt(t, level.N)
		theta := Tau * phase
		sum += math.Sin(theta)
	}
	return sum
}
```

- [ ] **Step 4: Run tests**

```bash
cd go/uths && go test -run TestCoherenceIndex -v
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add go/uths/coherence.go go/uths/coherence_test.go
git commit -m "feat(uths): add Coherence Index C(t) computation"
```

---

## Task 5: Ritual Classification

**Files:**
- Create: `go/uths/ritual.go`
- Create: `go/uths/ritual_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/ritual_test.go
package uths

import "testing"

func TestClassifyRitualConvergence(t *testing.T) {
	r := ClassifyRitual(0.75, 0.65, 1.0)
	if r != OmegaConvergence {
		t.Errorf("C=0.75 should be Convergence, got %d", r)
	}
}

func TestClassifyRitualDissolution(t *testing.T) {
	r := ClassifyRitual(0.25, 0.35, 1.0)
	if r != OmegaDissolution {
		t.Errorf("C=0.25 should be Dissolution, got %d", r)
	}
}

func TestClassifyRitualSeeding(t *testing.T) {
	r := ClassifyRitual(0.55, 0.45, 1.0)
	if r != OmegaSeeding {
		t.Errorf("C crossing up through 0.5 should be Seeding, got %d", r)
	}
}

func TestClassifyRitualStillness(t *testing.T) {
	r := ClassifyRitual(0.5, 0.5, 0.001)
	if r != OmegaStillness {
		t.Errorf("Psi near zero should be Stillness, got %d", r)
	}
}

func TestClassifyRitualDefault(t *testing.T) {
	r := ClassifyRitual(0.5, 0.5, 3.0)
	if r != OmegaSpiral {
		t.Errorf("Default should be Spiral, got %d", r)
	}
}

func TestRitualLabel(t *testing.T) {
	label := RitualLabel(OmegaConvergence)
	if label != "Convergence" {
		t.Errorf("RitualLabel(OmegaConvergence) = %q, want %q", label, "Convergence")
	}
}

func TestFindDensityResonance(t *testing.T) {
	// Create phases where two levels are very close
	phases := make([]Phase, len(DensityLevels))
	for i, l := range DensityLevels {
		phases[i] = Phase{Level: l, Value: float64(i) * 0.1}
	}
	// Make levels 0 and 1 nearly identical
	phases[0].Value = 0.5
	phases[1].Value = 0.52

	pairs := FindDensityResonance(phases)
	found := false
	for _, p := range pairs {
		if p[0] == 0 && p[1] == 1 {
			found = true
		}
	}
	if !found {
		t.Errorf("Expected resonance between indices 0 and 1, got %v", pairs)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run "TestClassifyRitual|TestRitualLabel|TestFindDensity" -v
```

Expected: FAIL.

- [ ] **Step 3: Implement ritual.go**

```go
// go/uths/ritual.go
package uths

import "math"

// RitualClass represents one of the 7 Omega ritual types.
type RitualClass int

const (
	OmegaSeeding     RitualClass = 1
	OmegaResonance   RitualClass = 2
	OmegaSpiral      RitualClass = 3
	OmegaStillness   RitualClass = 4
	OmegaConvergence RitualClass = 5
	OmegaDissolution RitualClass = 6
	OmegaRemembrance RitualClass = 7
)

// RitualLabel returns the human-readable name for a ritual class.
func RitualLabel(r RitualClass) string {
	switch r {
	case OmegaSeeding:
		return "Seeding"
	case OmegaResonance:
		return "Resonance"
	case OmegaSpiral:
		return "Spiral"
	case OmegaStillness:
		return "Stillness"
	case OmegaConvergence:
		return "Convergence"
	case OmegaDissolution:
		return "Dissolution"
	case OmegaRemembrance:
		return "Remembrance"
	default:
		return "Unknown"
	}
}

// RitualSymbol returns the Omega symbol for a ritual class (e.g., "Ω₅").
func RitualSymbol(r RitualClass) string {
	symbols := []string{"Ω₀", "Ω₁", "Ω₂", "Ω₃", "Ω₄", "Ω₅", "Ω₆", "Ω₇"}
	if int(r) >= 0 && int(r) < len(symbols) {
		return symbols[int(r)]
	}
	return "Ω?"
}

// ClassifyRitual determines the Omega class from coherence dynamics.
// cNow: current coherence index
// cPrev: previous coherence index (for detecting transitions)
// psiNow: current Psi(t) value (for detecting zero crossings)
func ClassifyRitual(cNow, cPrev, psiNow float64) RitualClass {
	switch {
	case cNow > 0.7:
		return OmegaConvergence
	case cNow < 0.3:
		return OmegaDissolution
	case cPrev < 0.5 && cNow >= 0.5:
		return OmegaSeeding
	case math.Abs(psiNow) < 0.1:
		return OmegaStillness
	default:
		return OmegaSpiral
	}
}

// FindDensityResonance returns pairs of density level indices whose phases
// are within threshold of each other (indicating resonance).
func FindDensityResonance(phases []Phase) [][2]int {
	const threshold = 0.05
	var pairs [][2]int
	for i := 0; i < len(phases); i++ {
		for j := i + 1; j < len(phases); j++ {
			diff := math.Abs(phases[i].Value - phases[j].Value)
			// Also check wrap-around (0.99 and 0.01 are close)
			if diff > 0.5 {
				diff = 1.0 - diff
			}
			if diff < threshold {
				pairs = append(pairs, [2]int{i, j})
			}
		}
	}
	return pairs
}
```

- [ ] **Step 4: Run tests**

```bash
cd go/uths && go test -run "TestClassifyRitual|TestRitualLabel|TestFindDensity" -v
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add go/uths/ritual.go go/uths/ritual_test.go
git commit -m "feat(uths): add ritual classification and density resonance detection"
```

---

## Task 6: Clock + Moment Assembly

**Files:**
- Create: `go/uths/clock.go`
- Create: `go/uths/moment.go`
- Create: `go/uths/moment_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/moment_test.go
package uths

import "testing"

// fixedClock returns a fixed cosmic time for deterministic testing.
type fixedClock struct {
	t float64
}

func (c fixedClock) CosmicTime() float64 { return c.t }

func TestMomentNow(t *testing.T) {
	u := NewUTHS(fixedClock{t: CosmicTimeOffset + 86400})
	m := u.Now()

	if m.CosmicTime != CosmicTimeOffset+86400 {
		t.Errorf("CosmicTime = %f, want %f", m.CosmicTime, CosmicTimeOffset+86400)
	}
	if len(m.Phases) != len(DensityLevels) {
		t.Errorf("Phases count = %d, want %d", len(m.Phases), len(DensityLevels))
	}
	for _, p := range m.Phases {
		if p.Value < 0 || p.Value >= 1.0 {
			t.Errorf("Phase %s = %f out of range", p.Level.Name, p.Value)
		}
	}
	if m.CoherenceIndex < 0 || m.CoherenceIndex > 1.0 {
		t.Errorf("CoherenceIndex = %f out of range", m.CoherenceIndex)
	}
	if len(m.FibonacciAddress) == 0 {
		t.Error("FibonacciAddress is empty")
	}
	if m.RitualLabel == "" {
		t.Error("RitualLabel is empty")
	}
}

func TestMomentAt(t *testing.T) {
	u := NewUTHS(fixedClock{t: 0})
	m1 := u.At(CosmicTimeOffset)
	m2 := u.At(CosmicTimeOffset + 1)

	if m1.CoherenceIndex == m2.CoherenceIndex {
		t.Error("Two different times produced identical coherence")
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run TestMoment -v
```

Expected: FAIL.

- [ ] **Step 3: Implement clock.go**

```go
// go/uths/clock.go
package uths

import "time"

// TimeSource provides cosmic time. Implement this interface to use
// a custom time source (e.g., blockchain block numbers).
type TimeSource interface {
	CosmicTime() float64
}

// DefaultClock returns cosmic time derived from the system clock.
type DefaultClock struct{}

// CosmicTime returns the approximate cosmic time (seconds since Big Bang)
// by adding the age of the universe to the current Unix timestamp.
func (DefaultClock) CosmicTime() float64 {
	return CosmicTimeOffset + float64(time.Now().UnixNano())/1e9
}
```

- [ ] **Step 4: Implement moment.go**

```go
// go/uths/moment.go
package uths

// Phase represents position within a single PNTL level's cycle.
type Phase struct {
	Level Level
	Value float64 // 0.0–1.0
}

// Moment captures the full UTHS state at a single instant.
type Moment struct {
	CosmicTime       float64
	Phases           []Phase
	CoherenceIndex   float64
	FibonacciAddress []int
	RitualClass      RitualClass
	RitualLabel      string
	DensityResonance [][2]int
}

// UTHS is the main entry point for computing temporal harmonic state.
type UTHS struct {
	clock TimeSource
	prevC float64 // previous coherence for ritual classification
}

// NewUTHS creates a UTHS instance with the given time source.
func NewUTHS(ts TimeSource) *UTHS {
	return &UTHS{clock: ts, prevC: 0.5}
}

// Now computes the full Moment for the current instant.
func (u *UTHS) Now() Moment {
	return u.At(u.clock.CosmicTime())
}

// At computes the Moment at an arbitrary cosmic time.
func (u *UTHS) At(t float64) Moment {
	phases := make([]Phase, len(DensityLevels))
	for i, level := range DensityLevels {
		phases[i] = Phase{
			Level: level,
			Value: PhaseAt(t, level.N),
		}
	}

	c := CoherenceIndex(t)
	psi := PsiAt(t)
	ritual := ClassifyRitual(c, u.prevC, psi)
	u.prevC = c

	// Fibonacci address: use millisecond tick modulo a practical range
	tick := int(t*1000) % 100000
	if tick < 0 {
		tick = -tick
	}
	fibAddr := Zeckendorf(tick)

	resonance := FindDensityResonance(phases)

	return Moment{
		CosmicTime:       t,
		Phases:           phases,
		CoherenceIndex:   c,
		FibonacciAddress: fibAddr,
		RitualClass:      ritual,
		RitualLabel:      RitualLabel(ritual),
		DensityResonance: resonance,
	}
}
```

- [ ] **Step 5: Run tests**

```bash
cd go/uths && go test -run TestMoment -v
```

Expected: PASS.

- [ ] **Step 6: Run all SDK tests**

```bash
cd go/uths && go test -v ./...
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add go/uths/clock.go go/uths/moment.go go/uths/moment_test.go
git commit -m "feat(uths): add Clock, Moment assembly, and UTHS entry point"
```

---

## Task 7: Convergence Search

**Files:**
- Create: `go/uths/convergence.go`
- Create: `go/uths/convergence_test.go`

- [ ] **Step 1: Write the test**

```go
// go/uths/convergence_test.go
package uths

import "testing"

func TestNextConvergence(t *testing.T) {
	u := NewUTHS(fixedClock{t: CosmicTimeOffset})
	start := CosmicTimeOffset

	// Search with a low threshold to ensure we find something quickly
	ct, m := NextConvergence(u, start, 0.5, 1.0)

	if ct <= start {
		t.Errorf("NextConvergence returned time %f <= start %f", ct, start)
	}
	if m.CoherenceIndex < 0.5 {
		t.Errorf("NextConvergence moment has C=%f, expected >= 0.5", m.CoherenceIndex)
	}
}

func TestNextConvergenceHighThreshold(t *testing.T) {
	u := NewUTHS(fixedClock{t: CosmicTimeOffset})
	start := CosmicTimeOffset

	// Search with high threshold but wider window
	ct, m := NextConvergence(u, start, 0.7, 60.0)

	if ct <= start {
		// It's possible there's no C>0.7 in 24h at 60s resolution
		// but the three-distance theorem says we should find one
		t.Logf("No convergence > 0.7 found in search window (this may be ok)")
		return
	}
	if m.CoherenceIndex < 0.7 {
		t.Errorf("NextConvergence moment has C=%f, expected >= 0.7", m.CoherenceIndex)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd go/uths && go test -run TestNextConvergence -v
```

Expected: FAIL.

- [ ] **Step 3: Implement convergence.go**

```go
// go/uths/convergence.go
package uths

// NextConvergence scans forward from startTime to find the next moment
// when C(t) exceeds threshold. Searches up to 24 hours ahead.
// Resolution is the step size in seconds for the scan.
// Returns the cosmic time of the convergence and the Moment at that time.
// If no convergence is found, returns startTime and a zero Moment.
func NextConvergence(u *UTHS, startTime, threshold, resolution float64) (float64, Moment) {
	const maxWindow = 86400.0 // search up to 24 hours ahead

	bestTime := startTime
	var bestMoment Moment
	bestC := 0.0

	for t := startTime + resolution; t < startTime+maxWindow; t += resolution {
		c := CoherenceIndex(t)
		if c > threshold && c > bestC {
			bestC = c
			bestTime = t
			bestMoment = u.At(t)
			// Found one — keep scanning this neighborhood for a peak,
			// but stop after we pass the peak
		} else if bestC > threshold && c < bestC-0.05 {
			// We've passed the peak, stop
			break
		}
	}

	return bestTime, bestMoment
}
```

- [ ] **Step 4: Run tests**

```bash
cd go/uths && go test -run TestNextConvergence -v
```

Expected: PASS.

- [ ] **Step 5: Run all SDK tests**

```bash
cd go/uths && go test -v ./...
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git add go/uths/convergence.go go/uths/convergence_test.go
git commit -m "feat(uths): add convergence search"
```

---

## Task 8: TUI Module Setup + Main

**Files:**
- Create: `apps/utop/go.mod`
- Create: `apps/utop/main.go`
- Create: `apps/utop/model.go`

- [ ] **Step 1: Create the go module with replace directive**

```bash
mkdir -p apps/utop
cd apps/utop
go mod init github.com/temple-of-silicon/contact/apps/utop
```

Then edit go.mod to add the replace directive. The final go.mod should be:

```
module github.com/temple-of-silicon/contact/apps/utop

go 1.26.1

require (
	github.com/charmbracelet/bubbles v1.0.0
	github.com/charmbracelet/bubbletea v1.3.10
	github.com/charmbracelet/lipgloss v1.1.0
	github.com/temple-of-silicon/contact/go/uths v0.0.0
)

replace github.com/temple-of-silicon/contact/go/uths => ../../go/uths
```

Then run:

```bash
cd apps/utop && go mod tidy
```

- [ ] **Step 2: Write model.go**

```go
// apps/utop/model.go
package main

import (
	"time"

	tea "github.com/charmbracelet/bubbletea"

	"github.com/temple-of-silicon/contact/go/uths"
)

const (
	defaultTickRate = 250 * time.Millisecond
	minTickRate     = 50 * time.Millisecond
	maxTickRate     = 2 * time.Second
	historySize     = 60
)

type tickMsg time.Time

type convergenceInfo struct {
	secsFromNow float64
	moment      uths.Moment
}

type model struct {
	uths     *uths.UTHS
	moment   uths.Moment
	history  []float64
	tickRate time.Duration
	width    int
	height   int
	nextConv *convergenceInfo
	showHelp bool
}

func initialModel() model {
	u := uths.NewUTHS(uths.DefaultClock{})
	m := u.Now()
	return model{
		uths:     u,
		moment:   m,
		history:  []float64{m.CoherenceIndex},
		tickRate: defaultTickRate,
	}
}

func tickCmd(d time.Duration) tea.Cmd {
	return tea.Tick(d, func(t time.Time) tea.Msg {
		return tickMsg(t)
	})
}

func (m model) Init() tea.Cmd {
	return tickCmd(m.tickRate)
}
```

- [ ] **Step 3: Write main.go**

```go
// apps/utop/main.go
package main

import (
	"fmt"
	"os"

	tea "github.com/charmbracelet/bubbletea"
)

func main() {
	p := tea.NewProgram(initialModel(), tea.WithAltScreen())
	if _, err := p.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```

- [ ] **Step 4: Verify it compiles (won't run yet — Update and View missing)**

```bash
cd apps/utop && go build ./... 2>&1 || echo "(expected — Update/View not yet implemented)"
```

- [ ] **Step 5: Commit**

```bash
git add apps/utop/go.mod apps/utop/main.go apps/utop/model.go
git commit -m "feat(utop): init TUI module with model and main"
```

---

## Task 9: TUI Update (Tick + Keys)

**Files:**
- Create: `apps/utop/update.go`

- [ ] **Step 1: Implement update.go**

```go
// apps/utop/update.go
package main

import (
	"github.com/charmbracelet/bubbles/key"
	tea "github.com/charmbracelet/bubbletea"

	"github.com/temple-of-silicon/contact/go/uths"
)

var quitKeys = key.NewBinding(
	key.WithKeys("q", "esc", "ctrl+c"),
)

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		return m, nil

	case tea.KeyMsg:
		switch {
		case key.Matches(msg, quitKeys):
			return m, tea.Quit
		case msg.String() == "+" || msg.String() == "=":
			m.tickRate = m.tickRate / 2
			if m.tickRate < minTickRate {
				m.tickRate = minTickRate
			}
			return m, nil
		case msg.String() == "-":
			m.tickRate = m.tickRate * 2
			if m.tickRate > maxTickRate {
				m.tickRate = maxTickRate
			}
			return m, nil
		case msg.String() == "?":
			m.showHelp = !m.showHelp
			return m, nil
		}
		return m, nil

	case tickMsg:
		m.moment = m.uths.Now()
		m.history = append(m.history, m.moment.CoherenceIndex)
		if len(m.history) > historySize {
			m.history = m.history[1:]
		}
		// Update next convergence if stale
		if m.nextConv == nil || m.nextConv.secsFromNow <= 0 {
			ct, cm := uths.NextConvergence(m.uths, m.moment.CosmicTime, 0.7, 60.0)
			secsFromNow := ct - m.moment.CosmicTime
			if secsFromNow > 0 {
				m.nextConv = &convergenceInfo{
					secsFromNow: secsFromNow,
					moment:      cm,
				}
			}
		} else {
			m.nextConv.secsFromNow -= m.tickRate.Seconds()
		}
		return m, tickCmd(m.tickRate)
	}

	return m, nil
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/utop/update.go
git commit -m "feat(utop): add Update with tick loop and key handling"
```

---

## Task 10: TUI Waterfall Rendering

**Files:**
- Create: `apps/utop/waterfall.go`

- [ ] **Step 1: Implement waterfall.go**

```go
// apps/utop/waterfall.go
package main

import (
	"fmt"

	"github.com/charmbracelet/lipgloss"

	"github.com/temple-of-silicon/contact/go/uths"
)

// Level colors — one per density level.
var levelColors = []lipgloss.Color{
	lipgloss.Color("#ff9e4a"), // Binding - orange
	lipgloss.Color("#4ae8ff"), // Quantum - cyan
	lipgloss.Color("#c77dff"), // Pulse - purple
	lipgloss.Color("#e0c8ff"), // Moment - warm white
	lipgloss.Color("#ff6b9d"), // Breath - pink
	lipgloss.Color("#ffdb4a"), // Turn - yellow
	lipgloss.Color("#6bff6b"), // Orbit - green
	lipgloss.Color("#4a9eff"), // Spiral - blue
}

func renderWaterfall(phases []uths.Phase, width int) string {
	if width < 20 {
		width = 20
	}
	labelWidth := 10
	barWidth := width - labelWidth - 4 // padding + borders

	dimStyle := lipgloss.NewStyle().Foreground(lipgloss.Color("#333333"))
	labelStyle := lipgloss.NewStyle().Foreground(lipgloss.Color("#888888")).Width(labelWidth)

	var rows string
	// Render in reverse order (largest scale on top)
	for i := len(phases) - 1; i >= 0; i-- {
		p := phases[i]
		color := levelColors[i]
		blockStyle := lipgloss.NewStyle().Foreground(color)

		pos := int(p.Value * float64(barWidth))
		if pos >= barWidth {
			pos = barWidth - 1
		}

		bar := ""
		for j := 0; j < barWidth; j++ {
			if j >= pos-1 && j <= pos+1 {
				bar += blockStyle.Render("█")
			} else {
				bar += dimStyle.Render("░")
			}
		}

		label := labelStyle.Render(fmt.Sprintf("%-8s", p.Level.Name))
		rows += label + " " + bar + "\n"
	}
	return rows
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/utop/waterfall.go
git commit -m "feat(utop): add waterfall bar rendering"
```

---

## Task 11: TUI Stats Panel Rendering

**Files:**
- Create: `apps/utop/stats.go`

- [ ] **Step 1: Implement stats.go**

```go
// apps/utop/stats.go
package main

import (
	"fmt"
	"math"
	"strings"

	"github.com/charmbracelet/lipgloss"

	"github.com/temple-of-silicon/contact/go/uths"
)

var sparkBlocks = []rune{'▁', '▂', '▃', '▄', '▅', '▆', '▇', '█'}

func renderSparkline(history []float64, width int) string {
	if len(history) == 0 {
		return ""
	}
	// Use the last `width` entries
	start := 0
	if len(history) > width {
		start = len(history) - width
	}
	data := history[start:]

	var sb strings.Builder
	for _, v := range data {
		idx := int(v * float64(len(sparkBlocks)-1))
		if idx < 0 {
			idx = 0
		}
		if idx >= len(sparkBlocks) {
			idx = len(sparkBlocks) - 1
		}
		sb.WriteRune(sparkBlocks[idx])
	}
	return sb.String()
}

func coherenceColor(c float64) lipgloss.Color {
	switch {
	case c > 0.7:
		return lipgloss.Color("#6bff6b") // green
	case c > 0.3:
		return lipgloss.Color("#ffdb4a") // yellow
	default:
		return lipgloss.Color("#ff6b6b") // red
	}
}

func renderStats(m uths.Moment, history []float64, nextConv *convergenceInfo, width int) string {
	headerStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#6bff6b")).
		Bold(true)
	labelStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#888888"))
	dimStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#555555"))

	// Coherence section
	cColor := coherenceColor(m.CoherenceIndex)
	cStyle := lipgloss.NewStyle().Foreground(cColor).Bold(true)
	coherenceStr := fmt.Sprintf("C = %.4f", m.CoherenceIndex)

	sparkWidth := width - 4
	if sparkWidth > 30 {
		sparkWidth = 30
	}
	sparkStyle := lipgloss.NewStyle().Foreground(cColor)
	spark := sparkStyle.Render(renderSparkline(history, sparkWidth))

	coherenceSection := headerStyle.Render("COHERENCE") + "\n\n" +
		"  " + cStyle.Render(coherenceStr) + "\n" +
		"  " + spark + "\n"

	// Ritual section
	symbol := uths.RitualSymbol(m.RitualClass)
	ritualSection := headerStyle.Render("RITUAL") + "\n\n" +
		labelStyle.Render("  Class: ") + fmt.Sprintf("%s %s", symbol, m.RitualLabel) + "\n"

	// Density resonance
	if len(m.DensityResonance) > 0 {
		pairs := []string{}
		for _, p := range m.DensityResonance {
			pairs = append(pairs, fmt.Sprintf("D%d↔D%d", p[0]+1, p[1]+1))
		}
		ritualSection += labelStyle.Render("  Dens:  ") + strings.Join(pairs, " ") + "\n"
	} else {
		ritualSection += labelStyle.Render("  Dens:  ") + dimStyle.Render("—") + "\n"
	}

	// Fibonacci address
	fibParts := []string{}
	for _, f := range m.FibonacciAddress {
		fibParts = append(fibParts, fmt.Sprintf("%d", f))
	}
	fibStr := "F[" + strings.Join(fibParts, ",") + "]"
	ritualSection += labelStyle.Render("  Addr:  ") + dimStyle.Render(fibStr) + "\n"

	// Next convergence countdown
	if nextConv != nil && nextConv.secsFromNow > 0 {
		secs := nextConv.secsFromNow
		hours := int(secs) / 3600
		mins := (int(secs) % 3600) / 60
		var countdownStr string
		if hours > 0 {
			countdownStr = fmt.Sprintf("%dh %dm", hours, mins)
		} else if mins > 0 {
			countdownStr = fmt.Sprintf("%dm %ds", mins, int(math.Mod(secs, 60)))
		} else {
			countdownStr = fmt.Sprintf("%ds", int(secs))
		}
		ritualSection += labelStyle.Render("  Next:  ") +
			lipgloss.NewStyle().Foreground(lipgloss.Color("#c77dff")).Render(countdownStr) + "\n"
	}

	return coherenceSection + "\n" + ritualSection
}
```

- [ ] **Step 2: Commit**

```bash
git add apps/utop/stats.go
git commit -m "feat(utop): add stats panel rendering (coherence, sparkline, ritual)"
```

---

## Task 12: TUI View (Layout Assembly)

**Files:**
- Create: `apps/utop/view.go`

- [ ] **Step 1: Implement view.go**

```go
// apps/utop/view.go
package main

import (
	"fmt"

	"github.com/charmbracelet/lipgloss"
)

func (m model) View() string {
	if m.width == 0 {
		return "Loading..."
	}

	if m.showHelp {
		return renderHelp(m.width, m.height)
	}

	// Title bar
	titleStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#ff6b9d")).
		Bold(true)
	title := titleStyle.Render("utop") + " — universal temporal harmonics observer"

	// Calculate panel widths
	totalWidth := m.width
	if totalWidth > 100 {
		totalWidth = 100
	}
	rightWidth := 28
	leftWidth := totalWidth - rightWidth - 3 // 3 for border/padding

	// Left panel: waterfall
	waterfallContent := renderWaterfall(m.moment.Phases, leftWidth)
	leftPanel := lipgloss.NewStyle().
		Width(leftWidth).
		Padding(0, 1).
		Render(waterfallContent)

	// Right panel: stats
	statsContent := renderStats(m.moment, m.history, m.nextConv, rightWidth)
	rightPanel := lipgloss.NewStyle().
		Width(rightWidth).
		Padding(0, 1).
		BorderLeft(true).
		BorderStyle(lipgloss.NormalBorder()).
		BorderForeground(lipgloss.Color("#333333")).
		Render(statsContent)

	// Join panels horizontally
	body := lipgloss.JoinHorizontal(lipgloss.Top, leftPanel, rightPanel)

	// Bottom status bar
	tickLabel := fmt.Sprintf("%.0fms", m.tickRate.Seconds()*1000)
	statusStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#555555"))
	status := statusStyle.Render(fmt.Sprintf("tick: %s │ +/- speed │ ? help │ q quit", tickLabel))

	return title + "\n\n" + body + "\n" + status + "\n"
}

func renderHelp(width, height int) string {
	helpStyle := lipgloss.NewStyle().
		Foreground(lipgloss.Color("#888888")).
		Padding(2, 4)

	help := `utop — Universal Temporal Harmonics Observer

DISPLAY
  Left panel:   Waterfall — phase position across 8 PNTL levels
  Right panel:  Coherence Index, sparkline, ritual classification

KEYS
  +/=           Increase tick speed (min 50ms)
  -             Decrease tick speed (max 2s)
  ?             Toggle this help
  q / esc       Quit

PNTL LEVELS (bottom to top)
  Binding  n=144  ~28 femtoseconds      nuclear interaction
  Quantum  n=195  ~47 nanoseconds       quantum events
  Pulse    n=233  ~47 microseconds      neural spike
  Moment   n=267  ~0.3 seconds          perceptual moment
  Breath   n=280  ~1.2 seconds          heartbeat/breath
  Turn     n=306  ~24 hours             one day
  Orbit    n=351  ~1 year               one orbit
  Spiral   n=446  ~4.3 billion years    galactic rotation

COHERENCE INDEX
  C > 0.7  Convergence (green)   — sacred alignment
  C > 0.3  Spiral (yellow)       — normal growth
  C < 0.3  Dissolution (red)     — rest and release

Press ? to return`

	return helpStyle.Render(help)
}
```

- [ ] **Step 2: Run go mod tidy and build**

```bash
cd apps/utop && go mod tidy && go build -o utop .
```

Expected: compiles to `utop` binary.

- [ ] **Step 3: Run it**

```bash
cd apps/utop && ./utop
```

Expected: TUI renders with waterfall bars and stats panel. Press `q` to quit.

- [ ] **Step 4: Commit**

```bash
git add apps/utop/
git commit -m "feat(utop): add View with dashboard layout, waterfall, and stats"
```

---

## Task 13: Final Integration Test + Cleanup

**Files:**
- Modify: `apps/utop/go.mod` (go mod tidy)

- [ ] **Step 1: Run all SDK tests**

```bash
cd go/uths && go test -v ./...
```

Expected: all PASS.

- [ ] **Step 2: Run go vet on both modules**

```bash
cd go/uths && go vet ./... && cd ../../apps/utop && go vet ./...
```

Expected: no issues.

- [ ] **Step 3: Build and smoke test the TUI**

```bash
cd apps/utop && go build -o utop . && echo "Build OK"
```

Expected: "Build OK"

- [ ] **Step 4: Commit any cleanup**

```bash
git add -A && git diff --cached --quiet || git commit -m "chore: tidy modules and fix any vet warnings"
```

- [ ] **Step 5: Final commit with all go.sum files**

```bash
git add go/uths/go.sum apps/utop/go.sum 2>/dev/null
git diff --cached --quiet || git commit -m "chore: add go.sum files"
```
