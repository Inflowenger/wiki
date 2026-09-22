# About this book

> **Status: outlined.** Thesis and section plan are committed; prose is not yet written.

## What this chapter will cover

- **Why this book exists.** The ecosystem spans a dozen repositories, each with good
  documentation for its own layer and no single place that explains how the layers make one
  argument. This book is that place.
- **What is open and what is not.** Infra and Fractal are closed source. Everything you
  write to build on the platform is open: `inflow-fusion`, the three plugin SDKs, the
  `inflowv1` protocol, the frontend packages, FloMorphic's API and canvas, the plugin
  catalog. The book documents the closed components by their **contracts** — REST
  endpoints, NATS subjects, guarantees — which is what you build against anyway.
- **How to read it.** Three routes: the evaluator's route (Parts 0, I, IV), the
  builder's route (Parts II, III, VI), and the operator's route (Parts I, VI).
- **Conventions.** Code is Go unless marked. Chapters that are outlined say so at the top.
  Where this book and a source repo disagree, the source repo wins.
- **The layering.** Each part is readable alone, but the parts are ordered as an argument:
  *here is a claim* (0) → *here is the machinery* (I–III) → *here is the claim surviving a
  real product* (IV) → *and a second one built on that* (V) → *and here is what it costs to
  run* (VI).

## Next

- [The thesis](the-thesis.md)
