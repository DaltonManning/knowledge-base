# Why the Interfaces Doc Was Built That Way

[← Back to index](README.md)

## Order of ideas
Definition → simplest working example → *why it matters* → rules → edge cases → comparison → reference list. This mirrors how experts actually learn a construct: see it, use it, understand its purpose, then harden the mental model with rules and exceptions.

## Definition before syntax
"A contract" is given before any code. A learner needs the *concept* anchored first, or the code becomes memorization instead of understanding.

## Minimal working example
`IShape` / `Circle` is deliberately trivial. A first example's job is to remove ambiguity about mechanics, not to demonstrate real-world complexity. Complexity comes later, once mechanics are automatic.

## "Why bother" placed early, not last
Most docs bury motivation at the end. Motivation was moved up because a rule is easier to retain once you know what problem it solves — polymorphism, testability, DI aren't abstract if you already care about the payoff.

## Table for syntax rules
Rules are terse, unrelated to each other, and referenced later rather than read linearly — a table beats prose for scanability and lookup speed.

## Explicit implementation and default methods as separate sections
Both are *exceptions to the basic model*, not core to it. Isolating them prevents diluting the core mental model with edge cases a beginner doesn't need on day one.

## Comparison table (interface vs abstract class)
This is the question every learner eventually asks. Answering it via direct comparison (not two separate essays) lets the reader map the concepts against each other in one glance — this is how decisions actually get made in practice.

## Built-in interfaces list
Purely for recognition — so when the learner sees `IDisposable` or `IEnumerable` elsewhere, it's not a mystery. Not meant to be exhaustive, just enough to anchor pattern recognition.

## "Where to go deeper" instead of exhaustive coverage
You said you'd Google the rest — so the doc ends by pointing at the *next* concepts rather than explaining them, respecting the stated scope instead of over-delivering.

## Overall principle
Teach the shape of the idea first, the exceptions second, and the comparisons last — because comparisons only mean something once both sides are understood.
