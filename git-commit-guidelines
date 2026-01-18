---

# Git Repository Guidelines

To maintain a clean and secure codebase, all contributors must follow the security protocols and commit message conventions outlined below.

## 🛡️ Security First

Before making your first commit, you **must** install the `ggshield` pre-commit hook. This prevents accidental leaks of secrets, API keys, or private credentials.

### Installation

Run the following command in your terminal at the root of the repository:

```bash
ggshield install --mode local

```

---

## 📝 Commit Message Conventions

We use **Semantic Commits** to categorize changes. This allows for automated changelog generation and easier repository navigation.

### Standard Semantic Prefixes

| Prefix | Use Case |
| --- | --- |
| **`feat:`** | Adding a new feature to the codebase. |
| **`fix:`** | Fixing a bug. |
| **`docs:`** | Changes to documentation only (e.g., README, inline comments). |
| **`style:`** | Formatting changes that do not affect code logic (white-space, semi-colons). |
| **`refactor:`** | Code changes that neither fix a bug nor add a feature. |
| **`perf:`** | Changes that improve application performance. |
| **`test:`** | Adding missing tests or correcting existing ones. |
| **`build:`** | Changes affecting the build system or external dependencies (e.g., npm, cargo). |
| **`ci:`** | Changes to CI configuration files and scripts (e.g., GitHub Actions). |
| **`chore:`** | Other changes that don't modify source or test files. |

### Structure of a Commit Message

A semantic commit message should follow this structure:

```text
<type>(<scope>): <short summary>

[optional body: explain the 'why' and 'how' of the change]

[optional footer: link issue IDs, e.g., "Closes #123"]

```

> **Example:** `feat(auth): add OAuth2 provider support for GitHub`

---

### Pro-Tip for your Homelab

Since you are developing on your **M4 Mac Mini** and deploying to the **K8 Plus (Talos)**, ensure your `feat:` and `fix:` messages are descriptive. This will make it much easier to track which version of your microservices is currently running in your production environment.

Would you like me to create a `.pre-commit-config.yaml` file to help automate these checks across your different development environments?