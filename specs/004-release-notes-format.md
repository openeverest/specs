# Release Notes Format Standardization

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2026-04-13
*   **Last Updated:** 2026-04-13
*   **Related Issues:**

---

## 1. Summary

Standardize the OpenEverest release notes format to improve readability and ensure consistency across both GitHub releases and the `openeverest/everest-doc` documentation repository. The key changes are: replacing tabbed highlight sections with plain sequential text, and establishing a shared template with a structured `Changes` parent section.

## 2. Motivation

Release notes are currently published in two places — GitHub releases and the everest-doc site — but the format is not consistent or formally specified. The documentation site renders highlights using tabs, which adds visual complexity and hides content behind interaction. Users must click tabs to discover highlights, and the tabbed layout does not translate meaningfully to GitHub's Markdown renderer.

A single, shared template eliminates duplication, reduces maintenance overhead, and ensures every reader — regardless of where they land — gets the same clear, linear structure.

## 3. Goals & Non-Goals

**Goals:**

*   Define one canonical release notes template used for both GitHub releases and everest-doc.
*   Replace tabbed highlight UI with plain sequential Markdown sections for better readability in all rendering contexts.
*   Define `Changes` as the required parent section grouping Added, Changed, Improved & Fixed entries.

**Non-Goals:**

*   Automation or tooling for generating release notes — out of scope.
*   Retroactively reformatting past release notes.
*   Changes to the release process or cadence.

## 4. Proposed Solution / Design

### Template structure

```markdown
# What's new in OpenEverest [Version]

➡️ **New to OpenEverest?** [Quickstart Guide](link)

> ⚠️ **Important Upgrade Notice**: [Breaking changes summary, if any.]

---

## 🌟 Release highlights

### [Highlight 1 title]
[Plain prose description. No tabs.]

### [Highlight 2 title]
[Plain prose description. No tabs.]

---

## 📝 Changes

### Added
- ...

### Changed
- ...

### Improved & Fixed
- ...

---

## 🚀 Upgrade to OpenEverest [Version]

### Using everestctl
...

### Using Helm directly
...

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/[prev]...[current]
```

### Key rules

1. **No tabs in highlights.** Each highlight is a plain `###` subsection under `## 🌟 Release highlights`. Prose, code blocks, and doc links are written sequentially.
2. **`Changes` is the required parent section** for all changelog entries. Subsections are `Added`, `Changed`, and `Improved & Fixed`. Every entry links to a GitHub issue or PR.
3. **One template, two destinations.** The same Markdown source is published to GitHub releases and committed to `openeverest/everest-doc`. No format divergence is permitted.

## 5. Definition of Done

*   Template is accepted and merged into this repository.
*   Template is adopted in `openeverest/everest-doc` for the next release cycle.
*   At least one release note is published using the new format on both GitHub and the docs site.

## 6. Alternatives Considered

**Keep tabs in the docs site only.** Rejected because it requires maintaining two diverging formats and hides content from users who don't interact with tabs.

**Rich custom components (callouts, cards).** Rejected for the same reason — Markdown portability to GitHub is a hard requirement.

## 7. Open Questions

*   Should the `Upgrade` section be optional for patch releases where no migration is needed?
*   Who owns the release notes template going forward — docs team or engineering?
