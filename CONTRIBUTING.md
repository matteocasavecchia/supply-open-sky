# Contributing to Supply Open Sky

Thank you for your interest in Supply Open Sky. This repository hosts the
public architectural documentation of the project. The implementation code
is maintained in a separate private repository during the current
development phase.

This document describes what kinds of contributions are welcome at this
stage, and how to submit them.

---

## What's in scope right now

Contributions to the public documentation are welcome, including:

- **Corrections** — typos, broken links, factual errors, inconsistencies
  between documents.
- **Clarifications** — passages that are ambiguous, confusing, or could be
  better explained.
- **Translations** — translations of public documents into other languages,
  particularly languages spoken in target deployment regions.
- **Feedback and questions** — open an issue if something is unclear, missing,
  or worth discussing. Questions from humanitarian operators, researchers,
  and engineers in adjacent fields are particularly welcome.

## What's not in scope right now

- **Code contributions.** The implementation code (drone firmware,
  scheduling engine, hub software, communication stack, simulator,
  dashboard) is maintained in a private repository during the current
  development phase. We are not accepting code contributions to the public
  repository at this time. This will change as the project matures.
- **New architectural proposals.** The current architecture is the result of
  ongoing design and field validation work. Substantial proposals to redesign
  parts of the system are best discussed first via an issue, before any
  documentation change.

## How to report an issue

Open a new issue at:
[github.com/matteocasavecchia/supply-open-sky/issues](https://github.com/matteocasavecchia/supply-open-sky/issues)

When reporting a documentation issue, please include:

- The document and section affected (e.g. `BLUEPRINT.md § 3.2`).
- A short description of the problem.
- If applicable, a suggested correction or clarification.

## How to propose a documentation change

We follow the standard fork-and-pull-request workflow:

1. **Fork** this repository to your GitHub account.
2. **Create a branch** with a descriptive name, e.g.
   `docs/fix-typo-blueprint-section-3` or `docs/add-italian-translation`.
3. **Make your changes** in your branch. Keep changes focused — one
   conceptual change per pull request.
4. **Commit** following the conventions described below.
5. **Open a pull request** against the `main` branch of this repository.
   Include a clear description of what changed and why.

### Commit message convention

This project follows
[Conventional Commits](https://www.conventionalcommits.org/) for commit
messages. The format is:

```
<type>: <short description>
```

Common types:

- `docs:` — documentation changes (most common in this repository).
- `chore:` — repository maintenance (e.g. `.gitignore`, GitHub configuration).
- `fix:` — corrections to factual errors or broken references.

Examples:

```
docs: fix typo in BLUEPRINT section 3.2
docs: add Italian translation of NOMENCLATURE
fix: correct broken link to LICENSE in README
```

Keep the short description in the imperative mood ("add X", not "added X")
and under 72 characters when possible.

## Code of Conduct

This project follows the [Contributor Covenant](./CODE_OF_CONDUCT.md) Code
of Conduct. By participating, you agree to abide by its terms.

## Security

If you believe you have found a security-relevant issue (in the
documentation, in any future code release, or in the project's infrastructure),
please **do not open a public issue**. See [SECURITY.md](./SECURITY.md) for
the responsible disclosure process.

## Tooling note

This project uses AI tooling in its development workflow for documentation
drafting and code review.

## License

By contributing to this repository, you agree that your contributions will
be licensed under the same
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](./LICENSE.md)
that covers the project documentation.
