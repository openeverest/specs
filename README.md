# OpenEverest Specs

This repository is the central hub for designing the future of the **OpenEverest** project. It contains **Specifications (Specs)** that propose, define, and archive significant changes, features, and architectural decisions for OpenEverest.

### 🎯 What This Repository Is For

*   **Major Proposals**: Capturing ideas, requirements, and designs for new features or substantial changes.
*   **Architecture & Design**: Documenting technical architecture, system design decisions, and the reasoning behind them.
*   **Transparent Roadmapping**: Providing a clear, historical record of *what* we plan to build, *why* we are building it, and *how* it will work.
*   **Community Collaboration**: Serving as the primary place for structured, asynchronous discussion on the project's direction.

### ❌ What This Repository Is *Not* For

*   **Bug Reports**: Please use the [main OpenEverest issue tracker](https://github.com/openeverest/openeverest/issues) for reporting bugs or problems.
*   **General Support Questions**: For usage help, please check our documentation or community channels.
*   **Code Implementation**: This repo is for `.md` specification files only. All implementation code belongs in the [main OpenEverest repository](https://github.com/openeverest/openeverest) or other designated code repos.

---

## 🚀 How to Use This Repository

The spec process bridges the gap between a high-level idea and concrete implementation. Here’s the workflow:

1.  **Identify an Idea**: Start with a conversation, usually as a GitHub Issue in the **[main OpenEverest repo](https://github.com/openeverest/openeverest/issues)**. This is where initial questions and scoping happen.
2.  **Create a Spec**: When an idea is substantial enough to require a formal design, create a new Specification (Spec) in this repository. **Link to the new spec from the original GitHub Issue** to provide detailed context and keep everyone informed.
3.  **Collaborate & Refine**: Discuss the spec through:
    *   **Async Comments**: Use GitHub's Pull Request review system to leave inline comments and suggestions.
    *   **Sync Meetings**: Bring the spec for discussion in our **bi-weekly community meetings** to align and make decisions.
4.  **Finalize & Implement**: Once the spec is approved and merged, it becomes the official guide for implementation work in the code repositories.

---

## 📝 Creating a New Specification

To ensure consistency and clarity, all specs follow a standard format.

### Naming and Numbering

*   Create a new markdown file in the specs folder of this repository.
*   Use sequential, zero-padded numbering: `###-descriptive-title.md`
*   **Example**: The first spec would be `001-plugin-architecture.md`

### Specification Template

Please copy the template [specs/###-spec-template.md](specs/###-spec-template.md) into your new file. Each section is designed to "work backwards" from the customer need to the technical solution, ensuring clarity on the problem and the definition of done.

## 🤝 Review & Discussion Process

1.  **Draft**: Create your spec and open a **Pull Request (PR)**. Set the status to `Draft`.
2.  **Review**:
    *   **Asynchronous**: Community members will review the PR, leaving comments and suggestions directly on the lines of the spec.
    *   **Synchronous**: Specs in `Draft` or `Under Review` status can be added to the agenda for our **bi-weekly community meetings** for live discussion and decision-making.
3.  **Revise**: Update the spec based on feedback, pushing new commits to the same PR branch.
4.  **Decision**: Once consensus is reached, a maintainer will:
    *   Change the status to `Accepted`.
    *   Merge the Pull Request.
    *   The spec is now an official project artifact.
5.  **Implementation**: The work defined in the spec can begin. Once all items in the **Definition of Done** are complete, update the spec's status to `Implemented`.

---

## 📄 License

All specifications in this repository are licensed under the **[Apache License 2.0](LICENSE)**.

---

## 💬 Getting Help

If you have questions about the spec process:
*   Join our community - see links at [openeverest.io](https://openeverest.io)

*Thank you for helping shape the future of OpenEverest!*
