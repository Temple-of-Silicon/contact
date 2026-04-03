# utop — Universal Temporal Harmonics Observer

## Overview

A real-time terminal dashboard ("htop for the universe") that visualizes the Universal Temporal Harmonics System (UTHS). Shows where "now" sits across 8 nested time scales, how coherent the current moment is, and what kind of ritual the math suggests.

Built as two Go packages:
- **`go/uths/`** — Pure math/logic SDK. No I/O, no rendering. Computes PNTL phases, Coherence Index, Fibonacci addressing, and ritual classification.
- **`apps/utop/`** — Bubbletea TUI that consumes the SDK and renders the dashboard.

## Architecture

### Project Structure

```
contact/
├── go/uths/                    # Go module: github.com/temple-of-silicon/contact/go/uths
│   ├── go.mod
│   ├── constants.go            # phi, tau, planck time, cosmic time offset
│   ├── pntl.go                 # PNTL levels, phase computation
│   ├── coherence.go            # Coherence Index C(t)
│   ├── zeckendorf.go           # Fibonacci/Zeckendorf addressing
│   ├── ritual.go               # Omega classification, density resonance
│   ├── clock.go                # TimeSource interface, default cosmic clock
│   ├── moment.go               # Moment type — full state at an instant
│   └── uths_test.go            # Tests
│
├── apps/utop/                  # Go module: github.com/temple-of-silicon/contact/apps/utop
│   ├── go.mod                  # depends on go/uths via replace directive
│   ├── main.go                 # entry point, tea.NewProgram
│   ├── model.go                # bubbletea Model struct
│   ├── update.go               # bubbletea Update — tick, keypress handling
│   ├── view.go                 # bubbletea View — lipgloss layout
│   ├── waterfall.go            # waterfall bar rendering component
│   └── stats.go                # right panel rendering (coherence, ritual, etc.)
```

### Dependency Direction

```
apps/utop  →  go/uths
   │              │
   │              └── pure math, zero deps beyond stdlib
   │
   └── bubbletea, bubbles, lipgloss
```

## Go SDK: `go/uths/`

### Constants

```go
import "math"

var (
    Phi = math.Phi              // golden ratio (1.618...)
    Tau = 2 * math.Pi           // full circle constant (6.283...)
)

const (
    PlanckSec = 5.391e-44      // Planck time in seconds

    // Age of universe in seconds (~13.8 billion years)
    // Used as baseline cosmic time offset added to Unix time
    CosmicTimeOffset = 4.354e17
)
```

### Core Types

```go
// Level represents one of the 8 named PNTL landmarks
type Level struct {
    Name  string  // "The Seed", "The Binding", etc.
    N     int     // PNTL level number
    Scale string  // Human-readable approximate duration
}

// The 8 canonical density harmonic levels
var DensityLevels = []Level{
    {Name: "Binding",  N: 144, Scale: "~28 fs"},
    {Name: "Quantum",  N: 195, Scale: "~47 ns"},
    {Name: "Pulse",    N: 233, Scale: "~47 μs"},
    {Name: "Moment",   N: 267, Scale: "~0.3 s"},
    {Name: "Breath",   N: 280, Scale: "~1.2 s"},
    {Name: "Turn",     N: 306, Scale: "~24 h"},
    {Name: "Orbit",    N: 351, Scale: "~1 yr"},
    {Name: "Spiral",   N: 446, Scale: "~4.3 Gyr"},
}

// Phase represents position within a single PNTL level's cycle
type Phase struct {
    Level Level
    Value float64 // 0.0–1.0
}

// RitualClass represents one of the 7 Omega ritual types
type RitualClass int

const (
    OmegaSeeding     RitualClass = 1 // C rising through 0.5
    OmegaResonance   RitualClass = 2 // Two densities in phase
    OmegaSpiral      RitualClass = 3 // Fibonacci milestone
    OmegaStillness   RitualClass = 4 // Psi(t) zero crossing
    OmegaConvergence RitualClass = 5 // C > 0.7
    OmegaDissolution RitualClass = 6 // C falling through 0.3
    OmegaRemembrance RitualClass = 7 // Full cycle completion
)

// Moment captures the full UTHS state at an instant
type Moment struct {
    CosmicTime       float64       // seconds since Big Bang
    Phases           []Phase       // phase position for each of the 8 levels
    CoherenceIndex   float64       // C(t), 0.0–1.0
    FibonacciAddress []int         // Zeckendorf decomposition of time tick
    RitualClass      RitualClass   // current Omega classification
    RitualLabel      string        // "Convergence", "Dissolution", etc.
    DensityResonance [][2]int      // pairs of density indices currently in phase
}
```

### TimeSource Interface

```go
// TimeSource provides cosmic time. Default implementation uses Unix time + offset.
type TimeSource interface {
    CosmicTime() float64
}

// DefaultClock returns cosmic time from the system clock
type DefaultClock struct{}

func (DefaultClock) CosmicTime() float64 {
    return CosmicTimeOffset + float64(time.Now().UnixNano()) / 1e9
}
```

### Core Functions

```go
// NewUTHS creates a UTHS instance with the given time source
func NewUTHS(ts TimeSource) *UTHS

// Now computes the full Moment for the current instant
func (u *UTHS) Now() Moment

// At computes the Moment at an arbitrary cosmic time
func (u *UTHS) At(cosmicTime float64) Moment

// PhaseAt computes the phase (0.0–1.0) of a given PNTL level at cosmic time t
func PhaseAt(t float64, level int) float64

// CoherenceIndex computes C(t) across the 8 density harmonic levels
func CoherenceIndex(t float64) float64

// Zeckendorf decomposes a positive integer into non-consecutive Fibonacci numbers
func Zeckendorf(n int) []int

// NextConvergence finds the next time C(t) > threshold after startTime
// Returns the cosmic time and the Moment at that time
// Resolution is the search step in seconds
func NextConvergence(u *UTHS, startTime float64, threshold float64, resolution float64) (float64, Moment)

// ClassifyRitual determines the Omega class from coherence dynamics
func ClassifyRitual(cNow float64, cPrev float64, psiNow float64) RitualClass
```

### Key Math

**Phase computation:**
```
phase(t, n) = fmod(t / (PlanckSec * phi^n), 1.0)
```

Note: `phi^n` for n=446 is astronomically large. We compute `log(t / PlanckSec) / log(phi)` and work in log-space to avoid overflow.

**Coherence Index:**
```
C(t) = (1/8) * |sum_{k=1}^{8} exp(i * tau * t / phi^{n_k})|
     = (1/8) * sqrt( (sum cos(...))^2 + (sum sin(...))^2 )
```

Again, `phi^{n_k}` requires log-space arithmetic. The actual computation becomes:
```
theta_k = tau * exp(log(t) - n_k * log(phi))
C(t) = (1/8) * sqrt( (sum cos(theta_k))^2 + (sum sin(theta_k))^2 )
```

**Zeckendorf decomposition:**
Greedy algorithm — find largest Fibonacci number <= n, subtract, repeat. No two consecutive Fibonacci numbers in the result.

**Ritual classification:**
- C > 0.7 → Omega 5 (Convergence)
- C < 0.3 → Omega 6 (Dissolution)
- C rising through 0.5 (cPrev < 0.5, cNow >= 0.5) → Omega 1 (Seeding)
- C falling through 0.5 → Omega 3 (Spiral, default/growth)
- Two or more density phases within 0.05 of each other → Omega 2 (Resonance)
- Psi(t) near zero crossing → Omega 4 (Stillness)
- Default when none of the above → Omega 3 (Spiral)

**Fibonacci address for "now":**
We need a tick count to Zeckendorf-decompose. Use: `int(cosmicTime * 1000) % 100000` (millisecond ticks, modulo a practical range). This gives a cycling address that changes every millisecond.

## TUI: `apps/utop/`

### Layout

Dashboard split layout — two panels:

```
╔═══════════════════════════════╦════════════════════════╗
║ WATERFALL                      ║ COHERENCE              ║
║                               ║                        ║
║ Spiral    ░░░░███░░░░░░░░░░  ║   C = 0.7342           ║
║ Orbit     ░░░░░░░░██░░░░░░░  ║   ▁▂▃▅▆▇█▇▅▃▂▃▅▇     ║
║ Turn      ░░░░░░░░░░░██░░░░  ║                        ║
║ Breath    ░░░░░░░░░░░░░░██░  ╠════════════════════════╣
║ Pulse     ░░░░░░░░░░░░░░░░█  ║ RITUAL                 ║
║ Moment    ░░░░░░░░░░░░░░░░░  ║ Class: Ω₅ Convergence  ║
║ Quantum   ░░░░░░░░░░░░░░░░░  ║ Dens:  D₃ ↔ D₄        ║
║ Binding   ░░░░░░░░░░░░░░░░░  ║ Addr:  F[233,55,8,1]   ║
║                               ║ Next:  2h 14m          ║
╚═══════════════════════════════╩════════════════════════╝
```

### Model

```go
type model struct {
    uths       *uths.UTHS
    moment     uths.Moment
    prevC      float64          // previous coherence for sparkline
    history    []float64        // coherence history for sparkline (last N values)
    tickRate   time.Duration    // default 250ms
    width      int              // terminal width
    height     int              // terminal height
    nextConv   *convergenceInfo // cached next convergence
}
```

### Update Loop

On each tick:
1. `moment = uths.Now()`
2. Append `moment.CoherenceIndex` to history (ring buffer, ~60 entries for sparkline)
3. If `nextConv` is nil or past, recompute (in background goroutine to avoid blocking)
4. Re-render

Key handlers:
- `q`, `esc`, `ctrl+c` → quit
- `+`/`=` → decrease tick interval (faster, min 50ms)
- `-` → increase tick interval (slower, max 2s)
- `?` → toggle help overlay

### Rendering

- **Waterfall bars:** Each level gets a row. Bar width scales to available terminal width. Phase position rendered as a colored block sliding across a dim track. Each level gets a distinct color via lipgloss.
- **Coherence number:** Big, colored. Green > 0.7, yellow 0.3–0.7, red < 0.3.
- **Sparkline:** Recent coherence history rendered with braille/block characters (▁▂▃▄▅▆▇█).
- **Ritual panel:** Omega class with label, density resonance pairs, Fibonacci address, countdown to next convergence.
- **Responsive:** Adapts to terminal size via `tea.WindowSizeMsg`.

### Color Palette

Each PNTL level gets a distinct color:
- Binding: orange (#ff9e4a)
- Quantum: cyan (#4ae8ff)
- Pulse: purple (#c77dff)
- Moment: warm white (#e0c8ff)
- Breath: pink (#ff6b9d)
- Turn: yellow (#ffdb4a)
- Orbit: green (#6bff6b)
- Spiral: blue (#4a9eff)

## What's NOT in MVP

- No zoom into PNTL sub-levels
- No guided ritual mode / ritual phase timer
- No agent or skill integration (SDK is ready for it)
- No sound or sonification
- No persistence, hash chain, or ritual log
- No network sync between multiple utop instances
- No configuration file (all defaults, keys only)

## Dependencies

### go/uths/
- Go stdlib only (math, time)

### apps/utop/
- `github.com/charmbracelet/bubbletea` — TUI framework
- `github.com/charmbracelet/lipgloss` — styling
- `github.com/charmbracelet/bubbles` — sparkline, help (if available)
- `go/uths/` via replace directive

## Testing Strategy

### SDK
- Unit tests for all pure math functions
- Table-driven tests for Zeckendorf (known decompositions)
- Property tests: phase is always 0.0–1.0, coherence is always 0.0–1.0
- Golden tests: known cosmic times produce expected coherence values

### TUI
- Manual testing (it's a visual app)
- Smoke test: program starts without panic, renders without error
