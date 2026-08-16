# Feature Evidence example for Swift

> Runtime evidence that connects **specifications to real execution**.

Feature Evidence is an instrumentation model for specification-driven software. It lets Swift code declare which specification or specification property it implements, and lets the runtime collect structured evidence that the implementation was actually exercised.

The goal is not to create another logging framework.

The goal is to close the gap between:

```text
Intent
  ↓
Specification
  ↓
Implementation
  ↓
Build
  ↓
Release
  ↓
Runtime execution
  ↓
Evidence
```

This project explores how that model can be implemented in Swift 6.2+ using macros and lightweight runtime instrumentation.

---

## Why?

Traditional observability answers:

> What happened while the program was running?

Code coverage answers:

> Which code was executed?

Tests answer:

> Which properties did our test suite verify?

Feature Evidence adds another question:

> **Which specification was actually exercised by this implementation?**

For example:

```swift
@Implements("player.resume")
func resume() {
    let position = currentPosition()
    play(from: position)
}
```

The annotation creates a machine-readable relationship:

```text
spec://player/resume
        │
        └── implemented-by
                │
                ▼
        Player.resume()
```

When the function executes, the runtime can produce evidence for that relationship.

---

## The core idea

There are three different concepts:

```text
Log
  = describes an event

Evidence
  = associates an event with a claim

Feature Passport
  = aggregates evidence into an attestable feature state
```

A log might say:

```text
Player.resume() called
```

Feature Evidence says:

```text
Specification:
    player.resume

Implementation:
    Player.resume

Execution:
    #8f31

Outcome:
    completed
```

The important difference is **semantic identity**.

The runtime event is not merely a string written by a developer. It is generated from a compile-time relationship between a specification and an implementation.

---

# Swift example

## Application code

A minimal application can look like this:

```swift
@Feature("player")
final class Player {

    @Implements("player.resume")
    func resume() {
        let position = currentPosition()
        play(from: position)
    }

    @Evidence("player.resume.preserves-position")
    func currentPosition() -> TimeInterval {
        42
    }

    func play(from position: TimeInterval) {
        print("Playing from \(position)s")
    }
}
```

Usage remains ordinary Swift:

```swift
let player = Player()
player.resume()
```

The developer does not explicitly write instrumentation calls.

---

# What the annotations mean

### `@Feature`

Associates a type with a feature:

```swift
@Feature("player")
final class Player {
    ...
}
```

Conceptually:

```text
Feature: player
    │
    └── Player
```

### `@Implements`

Declares that an implementation realizes a specification:

```swift
@Implements("player.resume")
func resume() {
    ...
}
```

Conceptually:

```text
Specification
    player.resume
        │
        └── implemented-by → Player.resume
```

### `@Evidence`

Associates an execution point with a specification property:

```swift
@Evidence("player.resume.preserves-position")
func currentPosition() -> TimeInterval {
    42
}
```

Conceptually:

```text
Property
    player.resume.preserves-position
        │
        └── observed-by → Player.currentPosition
```

---

# What the macro generates

The application developer writes:

```swift
@Implements("player.resume")
func resume() {
    let position = currentPosition()
    play(from: position)
}
```

The macro can conceptually transform this into:

```swift
func resume() {
    FeatureEvidence.enter(
        spec: "player.resume",
        implementation: "Player.resume"
    )

    defer {
        FeatureEvidence.exit(
            spec: "player.resume",
            implementation: "Player.resume"
        )
    }

    let position = currentPosition()
    play(from: position)
}
```

Likewise:

```swift
@Evidence("player.resume.preserves-position")
func currentPosition() -> TimeInterval {
    42
}
```

can be instrumented conceptually as:

```swift
func currentPosition() -> TimeInterval {
    let result = 42

    FeatureEvidence.observe(
        property: "player.resume.preserves-position",
        implementation: "Player.currentPosition"
    )

    return result
}
```

The exact macro implementation is an implementation detail. The important contract is the relationship between the source declaration and the resulting evidence.

---

# Runtime evidence

A minimal runtime collector could expose an API like:

```swift
enum FeatureEvidence {

    static func enter(
        spec: StaticString,
        implementation: StaticString
    ) {
        // Record execution start.
    }

    static func exit(
        spec: StaticString,
        implementation: StaticString
    ) {
        // Record execution completion.
    }

    static func observe(
        property: StaticString,
        implementation: StaticString
    ) {
        // Record specification-property evidence.
    }
}
```

A production implementation would normally avoid `print` and send structured events to an evidence collector.

For example:

```swift
struct EvidenceEvent {
    let kind: Kind
    let specificationID: String
    let implementationID: String
    let executionID: UUID

    enum Kind {
        case entered
        case completed
        case failed
        case observed
    }
}
```

The resulting data is no longer an unstructured log line. It is a typed event that can be associated with the specification graph.

---

# From execution to Feature Passport

Suppose the application executes:

```swift
player.resume()
```

The runtime might produce:

```text
player.resume
    │
    ├── implementation: Player.resume
    │
    ├── execution: #8f31
    │
    └── property evidence:
            player.resume.preserves-position
```

SpecGraph can then materialize a Feature Passport:

```yaml
feature: player
release: 5.3.0

claims:
  player.resume:
    implementation: verified
    execution: observed

  player.resume.preserves-position:
    evidence: observed
    implementation: Player.currentPosition
```

The passport is not the raw telemetry.

It is a higher-level representation of what can be established from the available evidence.

---

# Evidence is not logging

Feature Evidence may use logging, signposts, tracing, or another observability backend.

That does not make the concepts equivalent.

The distinction is:

```text
Logs
    "What happened?"

Evidence
    "What claim does this execution support?"

Feature Passport
    "What can we attest about this feature?"
```

A useful rule is:

> **Logs describe executions. Evidence tells you what they prove.**

Feature Evidence therefore sits above the transport mechanism.

The same evidence could be transported through:

- an in-memory collector;
- OSLog;
- signposts;
- a test collector;
- an OpenTelemetry backend;
- a custom telemetry service.

The transport is replaceable. The specification relationship is not.

---

# Specification-aware coverage

Traditional code coverage can tell us:

```text
Player.swift:42 was executed
```

Feature Evidence can tell us:

```text
spec://player/resume
    ↓
Player.resume
    ↓
executed in release 5.3.0
```

This enables a different kind of coverage:

> **Specification coverage** — which parts of the declared specification graph have runtime evidence?

For example:

```text
Specification                         Evidence

player.resume                         ✓ executed
player.resume.preserves-position     ✓ observed
player.resume.handles-expired-token  ? no evidence
player.resume.restores-audio         ✗ failed
```

This is much closer to the questions asked by a specification-driven development system than ordinary code coverage.

---

# Tests as evidence

The same model can connect tests to specifications:

```swift
@Test
@Verifies("player.resume.preserves-position")
func resumePreservesPosition() async {
    // ...
}
```

Now the graph contains two independent evidence sources:

```text
                    Specification
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       Implementation              Test
              │                     │
              ▼                     ▼
       Runtime Evidence       Verification Evidence
              │                     │
              └──────────┬──────────┘
                         ▼
                  Feature Passport
```

This allows a specification to distinguish between:

- declared implementation;
- statically present implementation;
- tested implementation;
- shipped implementation;
- executed implementation;
- observed property;
- verified property.

---

# Evidence levels

A useful model is to treat evidence as progressive rather than boolean.

```text
Declared
    ↓
Compiled
    ↓
Linked
    ↓
Shipped
    ↓
Executed
    ↓
Observed
    ↓
Verified
```

For example:

```text
player.resume

Declared       ✓
Compiled       ✓
Shipped        ✓
Executed       ✓
Preserved      ✓
Verified       ✓
```

This allows a Feature Passport to represent the actual state of a feature instead of simply saying:

```text
implemented = true
```

---

# Specification IDs

In a real SpecGraph integration, specification IDs should preferably not be handwritten strings.

Instead of:

```swift
@Implements("player.resume")
```

a generated Swift SDK could expose typed identifiers:

```swift
enum PlayerSpec {
    static let resume = SpecID("...")
    static let resumePreservesPosition = SpecID("...")
}
```

Then:

```swift
@Implements(PlayerSpec.resume)
func resume() {
    ...
}

@Evidence(PlayerSpec.resumePreservesPosition)
func currentPosition() -> TimeInterval {
    ...
}
```

This creates a useful pipeline:

```text
SpecGraph
    │
    │ generates
    ▼
Specification SDK
    │
    ▼
Swift Compiler
    │
    ▼
Macro instrumentation
    │
    ▼
Runtime Evidence
    │
    ▼
Feature Passport
```

It also turns specification identity into a compiler-visible dependency rather than an arbitrary string.

---

# Swift architecture

The project can be split into a few small components:

```text
┌──────────────────────────────┐
│          SpecGraph           │
│                              │
│ Specification / Property     │
└──────────────┬───────────────┘
               │
               │ specification IDs
               ▼
┌──────────────────────────────┐
│     SpecificationKit        │
│                              │
│ @Feature                     │
│ @Implements                  │
│ @Evidence                    │
│ @Verifies                    │
└──────────────┬───────────────┘
               │
               │ compile-time
               ▼
┌──────────────────────────────┐
│       Swift Macro Plugin     │
└──────────────┬───────────────┘
               │
               │ instrumentation
               ▼
┌──────────────────────────────┐
│      FeatureEvidence         │
│                              │
│ execution events             │
│ execution context            │
│ provenance                   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Feature Passport       │
│                              │
│   attestable feature state   │
└──────────────────────────────┘
```

---

# Design principles

## 1. Compile-time identity, runtime evidence

The relationship between specification and implementation should be established at compile time.

Runtime instrumentation should only record what happened.

```text
Compile time:
    Specification ↔ Implementation

Runtime:
    Implementation → Execution → Evidence
```

---

## 2. Evidence is structured

Avoid making the evidence model dependent on log messages.

Prefer:

```swift
EvidenceEvent(
    kind: .completed,
    specificationID: ...,
    implementationID: ...,
    executionID: ...
)
```

over:

```text
"Player.resume completed"
```

---

## 3. Instrumentation must be optional

A production application should be able to select different evidence modes:

```text
off
debug
test
sampled
production
full
```

The evidence system should have a predictable and measurable overhead.

---

## 4. Runtime evidence is not proof of arbitrary correctness

Execution alone does not prove that an implementation is correct.

For example:

```text
function executed
```

does not imply:

```text
specification satisfied
```

Evidence must therefore distinguish between:

```text
executed
observed
asserted
verified
```

This prevents the system from turning telemetry into unjustified claims.

---

# Possible future extensions

This repository is intentionally focused on the smallest useful primitive. Several extensions are possible:

### Async execution context

Propagate the active feature/specification context across Swift concurrency boundaries:

```text
FeatureExecutionContext
    ├── feature
    ├── specification
    ├── execution ID
    └── trace ID
```

### OSLog / signposts

Use Apple's instrumentation infrastructure as one backend while keeping Feature Evidence semantically independent from logging.

### Test integration

Generate verification evidence from Swift Testing.

### Build provenance

Bind implementation evidence to:

```text
source
    ↓
commit
    ↓
build artifact
    ↓
release
```

### Runtime attestation

Produce signed evidence or Feature Passports suitable for remote verification.

### SpecGraph integration

Store runtime evidence directly as nodes and relationships in the specification graph.

---

# Relationship to SpecGraph

Feature Evidence is intended to be a runtime extension of a specification graph.

A complete chain looks like:

```text
Intent
  ↓
Requirement
  ↓
Property
  ↓
Specification
  ↓
Implementation
  ↓
Test
  ↓
Artifact
  ↓
Release
  ↓
Runtime
  ↓
Evidence
```

This turns the specification graph from a primarily design-time artifact into a system that can continuously accumulate evidence from real software execution.

The long-term goal is to make questions such as these machine-answerable:

```text
Which specifications are implemented?
Which implementations shipped?
Which features were actually exercised?
Which specification properties have runtime evidence?
Which released features have no evidence?
Which evidence came from tests versus production?
What changed between two releases?
```

---

# Status

This project is currently an exploration/prototype of the **Feature Evidence** model.

The initial goal is deliberately small:

1. Define stable specification identities.
2. Attach specification identities to Swift declarations.
3. Generate instrumentation using Swift macros.
4. Collect structured runtime evidence.
5. Build a minimal Feature Passport from that evidence.

The important experiment is not the logging backend.

It is whether a language-level connection between **Specification → Implementation → Execution → Evidence** can become a useful primitive for specification-driven software engineering.

---

# License

TBD.

---

## One sentence

> **Feature Evidence is specification-aware runtime instrumentation: it connects what the software is declared to implement with what actually happened when the software ran.**