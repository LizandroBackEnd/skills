---
name: git-angular-commit-convention
description: Universal Git workflow and Angular commit message convention skill. Directs the creation of structured, semantic, and automated-release-friendly commit messages and branch strategies based on the Angular specification and Conventional Commits. Triggers on "git commit", "commit message", "changelog", "conventional commits", "git workflow", "semantic versioning".
license: MIT
metadata:
 author: lizdev
 version: "1.0.0"
---

# UNIVERSAL GIT WORKFLOW AND ANGULAR COMMIT CONVENTION

This technical skill defines a unified Git commit convention and release automation contract based on the Angular Commit Message Guidelines, the Conventional Commits Specification, and Semantic Versioning 2.0.0. Compliance ensures clear commit histories, automated changelog generation, and predictable version increments.

---

## 1. COMMIT STRUCTURE AND ANATOMY

Every commit message must strictly adhere to a structured three-part layout: a Header, an optional Body, and an optional Footer. Each section is separated by exactly one blank line.

```text
<type>(<scope>): <short summary>
<BLANK LINE>
[optional body]
<BLANK LINE>
[optional footer(s)]
```

### Header Specifications
* Format: `<type>(<scope>): <short summary>`.
* Character Limit: The entire header line must not exceed 100 characters to ensure readability across Git tools and hosting interfaces. To preserve maximum visual clarity in terminal graphs, keep the summary between 50 and 72 characters where possible.
* Lowercase Rule: The type and scope must be written entirely in lowercase.
* Summary Content: 
 * Use the imperative, present tense: "change" instead of "changed" or "changes".
 * Do not capitalize the first letter of the summary.
 * Do not append a trailing period or any terminal punctuation at the end.

### Body Specifications
* Format: The body must start exactly one blank line below the header.
* Wrapping: Any line within the commit body must not exceed 100 characters.
* Grammar: Write in the imperative, present tense.
* Content: The body must describe the motivation behind the change and provide a clear contrast with the previous behavior. Do not explain *how* the change was made; explain *why* it was necessary and *what* it fixes.

### Footer Specifications
* Format: The footer must begin exactly one blank line below the body.
* Tokens: Each footer entry must consist of a word token, followed by a separator (either `:<space>` or `<space>#`), followed by a string value. Use a hyphen (`-`) instead of spaces for multi-word tokens (e.g., `Reviewed-by`). The only exception is the `BREAKING CHANGE` token, which is also valid with spaces.
* Breaking Changes: Must be explicitly declared here using the uppercase token `BREAKING CHANGE:` or `BREAKING-CHANGE:` followed by a description of what broke and how to migrate.
* Issue Referencing: Closes or references specific issues using the appropriate Git keywords followed by the issue number (e.g., `Closes #123`).

---

## 2. ALLOWED TYPES AND SCOPE DEFINITIONS

To maintain a clean and parseable commit history, only the following standardized types are permitted:

| Type | Intended Usage | Impact on SemVer |
| :--- | :--- | :--- |
| feat | Introduces a brand new feature or backward-compatible API modification. | MINOR |
| fix | Patches an existing bug or corrects incorrect internal behavior. | PATCH |
| docs | Changes documentation files exclusively without modifying any code. | None |
| style | Changes formatting, whitespace, or semi-colons without affecting code execution. | None |
| refactor | Modifies code without fixing a bug or adding a feature (e.g., renaming variables). | None |
| perf | Alters code specifically to improve system performance. | None |
| test | Adds missing tests, restructures testing setups, or corrects existing specs. | None |
| build | Modifies build configurations, package scripts, or external dependencies. | None |
| ci | Edits CI/CD pipeline files, workflows, and infrastructure scripts. | None |
| chore | Auxiliary changes that do not modify source files or testing frameworks. | None |
| revert | Reverts a previous commit. Must reference the reverted commit's SHA. | None |

### Scope Conventions
The scope is optional but recommended to provide contextual information about where the change occurred. It must consist of a lowercase noun surrounded by parentheses. 
* Scope Casing: Always use lowercase.
* Granular Scopes: Define scopes based on application modules, domain structures, or library components (e.g., `auth`, `parser`, `api`, `routing`).
* Multi-module Scopes: If a change spans multiple modules or components, use `*` as the scope. If the change affects too many areas, consider splitting it into separate, highly organized commits.

---

## 3. BREAKING CHANGES AND SEMANTIC VERSIONING INTEGRATION

The commit history acts as the direct upstream source of truth for Semantic Versioning (SemVer) version increments. Continuous Integration (CI) systems read these commit patterns to automatically bump version numbers.

### Signaling a Breaking Change
A breaking change is defined as any backward-incompatible change introduced to the public API. It must be flagged using one of the following two methods:

1. The Exclamation Mark Indicator (`!`):
 An exclamation mark is placed immediately after the type/scope and before the colon.
 ```text
 feat(auth)!: replace sessions with JWT authentication
 ```
2. The Footer Declaration (`BREAKING CHANGE:`):
 A `BREAKING CHANGE:` or `BREAKING-CHANGE:` token is added in uppercase at the very beginning of the footer section, followed by a space, and a description of the breaking changes and migration path.
 ```text
 BREAKING CHANGE: the old session endpoint has been removed. Use /login instead.
 ```

*Combining both methods* is highly recommended to maximize readability in both git log summaries and full release changelogs.

### Semantic Versioning Increments
Version numbers are formatted as `MAJOR.MINOR.PATCH`. Commit types are mapped directly to these increments:

1. MAJOR Increment (`X.0.0`):
 Triggered by *any* commit type that contains a breaking change flag (either `!` or a `BREAKING CHANGE` footer). This indicates backward-incompatible API changes.
2. MINOR Increment (`x.Y.0`):
 Triggered by commits of type `feat` that do *not* contain breaking changes. This indicates new, backward-compatible functionality introduced to the public API.
3. PATCH Increment (`x.y.Z`):
 Triggered by commits of type `fix` that do *not* contain breaking changes. This indicates backward-compatible internal bug fixes.

Other commit types (such as `docs`, `refactor`, `style`, `test`) do not trigger a version bump unless specifically configured in custom release pipelines.

---

## 4. STEP-BY-STEP WORKFLOW: WRITING AND REFACTORING COMMITS

This step-by-step algorithm must be followed by engineers or automated tools when preparing and formatting commit messages:

### Step 1: Stage and Analyze the Diff
Analyze all staged modifications by running a targeted git diff.
```bash
git diff --cached
```
Identify if the modifications are highly focused. If the staged changes span multiple unrelated features, refactorings, and bug fixes, divide the changes and stage them progressively to commit them as separate, atomic units.

### Step 2: Classify the Primary Change Type
Determine the primary impact of the staged code to choose the correct type:
* If it introduces a new feature or backward-compatible API: `feat`.
* If it resolves a bug or corrects unexpected runtime behavior: `fix`.
* If it changes only configurations, build scripts, or pipelines: `build` or `ci`.
* If it updates text descriptions or markdown files: `docs`.

### Step 3: Define the Scope and Check for Breaking Changes
* Locate the specific subsystem or module that was changed to define the scope (e.g., `parser`, `ui`, `core`).
* Assess if any modified code breaks backward compatibility with existing consumers of the public API. If so, flag the commit header with an exclamation mark (`!`) and prepare a detailed migration path for the footer.

### Step 4: Draft, Limit, and Refactor the Message
Write the message in an editor or terminal while enforcing strict constraints:
* Verify the header is under 100 characters.
* Ensure the summary is written in the imperative, present tense ("add", not "added") and lacks trailing punctuation.
* If a body is included, separate it from the header with exactly one blank line.
* If a footer is included, format the tokens cleanly (e.g., use `Closes #123` to reference issues).

---

## 5. CONCRETE COMMIT EXAMPLES

### a) New Feature Addition
Focuses on introducing a backward-compatible array-parsing capability to the query utility module.

#### INCORRECT / SILLY / UNSTRUCTURED
```text
added some array parsing stuff to query util i hope it works
```

#### CORRECT / ANGULAR CONVENTIONAL
```text
feat(query): add support for parsing comma-separated arrays

Implement a custom splitter to parse bracketed string queries into
native JavaScript arrays.

Closes #84
```

### b) Bug Fix Referencing an Issue
Addresses a specific racing condition within the HTTP client and closes a tracked ticket.

#### INCORRECT / SILLY / UNSTRUCTURED
```text
Fixed the crash when requests are fired too fast. Resolves ticket 422!
```

#### CORRECT / ANGULAR CONVENTIONAL
```text
fix(http): prevent racing conditions on rapid duplicate requests

Introduce a request identifier and track the latest active request.
Dismiss all incoming responses that do not match the latest request ID.

Closes #422
```

### c) Critical Breaking Change Altering a Public API
Drops support for an outdated, insecure authentication method and provides migration steps.

#### INCORRECT / SILLY / UNSTRUCTURED
```text
remove sessions because JWT is better. this will break old logins!
```

#### CORRECT / ANGULAR CONVENTIONAL
```text
feat(auth)!: remove cookie-based session authentication

Remove all cookie-based session endpoints and middleware. All clients
must now authenticate using the new Bearer Token protocol.

BREAKING CHANGE: Session authentication cookies are no longer supported.
Clients must modify their request headers to include 'Authorization: Bearer <token>'.
```

### d) Multi-line Commit with Detailed Motivation
Provides deep architectural context explaining why a specific rendering method was refactored.

#### INCORRECT / SILLY / UNSTRUCTURED
```text
refactored the rendering loop for performance. it should be faster now.
```

#### CORRECT / ANGULAR CONVENTIONAL
```text
refactor(render): optimize element reconciliation loop

Introduce a double-ended diffing algorithm to minimize DOM operations.
This replaces the naive linear reconciliation which performed redundant
removals and insertions.

This change reduces typical re-rendering times by 40% on large lists.
```

---

## 6. AUTOMATION AND CONFIGURATION PRESETS

To guarantee compliance with the Angular commit guidelines and Conventional Commits specification, you must lint commit messages locally and within your CI pipelines.

### Commitlint Configuration (`commitlint.config.js`)
Install the required packages as development dependencies:
```bash
npm install --save-dev @commitlint/cli @commitlint/config-angular
```

Create a `commitlint.config.js` or `commitlint.config.mjs` file in the root of your project to extend the Angular ruleset:

```javascript
export default {
 extends: ['@commitlint/config-angular'],
 rules: {
 'type-enum': [
 2,
 'always',
 [
 'feat',
 'fix',
 'docs',
 'style',
 'refactor',
 'perf',
 'test',
 'build',
 'ci',
 'chore',
 'revert'
 ]
 ],
 'type-case': [2, 'always', 'lowercase'],
 'scope-case': [2, 'always', 'lowercase'],
 'subject-empty': [2, 'always', 'false'],
 'subject-full-stop': [2, 'never', '.'],
 'header-max-length': [2, 'always', 100]
 }
};
```

### Local Enforcement with Husky
Husky enforces valid commit messages locally by executing commitlint during the Git commit lifecycle.

1. Initialize Husky in your project repository:
 ```bash
 npx husky init
 ```

2. Configure Husky to lint the commit message before the commit is finalized:
 Create or modify the `.husky/commit-msg` hook file to execute the lint command:
 ```bash
 echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
 chmod +x .husky/commit-msg
 ```

Any attempt to commit a message that violates the Angular commit structure will be rejected immediately, preventing non-compliant commits from ever reaching your remote branches.