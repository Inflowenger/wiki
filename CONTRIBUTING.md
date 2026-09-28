# Contributing

This book is assembled from the Inflowenger ecosystem's source repositories. It is a
*synthesis*, not a second source of truth.

## The rule that governs everything

> **Where this book and a source repository disagree, the source repository wins.**

If you find a contradiction, the fix belongs here, not there. Open an issue naming the
chapter and the normative source. The
[normative sources table](book/99-appendix/repositories.md#normative-sources) lists which
repo owns which subject.

## Chapter states

Every chapter declares its state at the top:

| State | Means |
| --- | --- |
| *(none)* | Written. Prose is complete and reviewed. |
| **outlined** | Thesis and section plan are committed; prose is not yet written. |

An outlined chapter is a commitment to content, not content. Filling one in is the most
useful contribution available — the outline tells you what it must establish and which
sources to write it from.

## Writing conventions

- **Self-contained.** A chapter explains its subject well enough to be read standalone.
  Some overlap with source repo docs is accepted — this is a book, and a book that requires
  twelve browser tabs is not one.
- **Cite sources at the end**, under **Source material**, pointing at the normative repo doc.
- **Code is Go** unless marked otherwise. Illustrative code that is not a shipped package
  says so explicitly (see
  [Compiling a YAML DSL](book/02-fusion/compiling-a-yaml-dsl.md)).
- **State the limits.** Every chapter that makes a claim should name what would falsify it,
  or what it costs. The
  [awkward cases](book/02-fusion/coverage.md#the-awkward-cases-stated-plainly) section is
  the model.
- **Closed components are documented by contract.** Infra and Fractal are not open source.
  Document endpoints, subjects and guarantees — never speculate about internals.
- **Relative links only**, so the book builds as a static site, a PDF and a GitHub repo
  from one source.

## Structure

```
book/
├── 00-preface/      the thesis and how to read
├── 01-platform/     what Inflowenger is
├── 02-fusion/       any source format → a running flow   ← written
├── 03-plugins/      inflowv1 and the SDKs
├── 05-flomorphic/   the runtime's first product
├── 06-venapce/      a product built on the product
├── 07-architecture/ scale, isolation, customization
└── 99-appendix/     maps and further reading
```

Add a chapter by creating the file, adding a row to its part's `index.md` table, and adding
a line to [SUMMARY.md](SUMMARY.md) — which is the book's spine for any static-site or PDF
build.

## License

Contributions are accepted under [Apache 2.0](LICENSE).
