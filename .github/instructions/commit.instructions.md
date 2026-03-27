# Commit Message Rules (Atomic Commits with Conventional Commits)

When generating commit messages:

- Always use Conventional Commits format.
- Format: `<emoji> <type>(optional-scope): <short reason>`
- The message must be written in English.
- Provide only ONE clear and specific reason.
- **Always include the emoji** at the start of the commit message.
- Keep it concise (max 10–12 words after the colon).
- Do not add descriptions, bullet points, or multiple sentences (unless body is explicitly needed).
- Use lowercase after the colon.

---

## 1. Commit Structure

The commit message must strictly follow this structure:

```

<emoji> <type>[optional scope]: <description>

[optional body]

[optional footer(s)]

```

### Type (Required)

Communicates the intention of the change:

- 💻 feat: introduces a new feature
- 🛠️ fix: fixes a bug
- 📗 docs: documentation changes
- ⚙️ refactor: code change without feature or fix
- 🧪 test: adds or updates tests
- 🔨 build: build system or dependencies
- 🔁 ci: CI/CD configuration
- ⚡ perf: performance improvements

### Scope (Optional)

- A noun in parentheses describing the affected area.
- Example: `(auth)`, `(api)`, `(parser)`

### Description (Required)

- Short, clear summary of the change.
- Must follow `<type>(scope): description`
- Always lowercase and concise.

### Body (Optional)

- Add only if necessary.
- Must be separated by **one blank line**.
- Can include multiple paragraphs.

### Footer (Optional)

- Used for references or breaking changes.
- Must be separated by **one blank line**.

Examples:
- `Refs: #123`
- `BREAKING CHANGE: authentication flow updated`

### Breaking Changes

- Add `!` before the colon:
  
```

💻 feat(api)!: change authentication response structure

````

---

## 2. Atomic Commit Principles (STRICT)

- Each commit must represent **a single logical change**.
- Prefer **many small commits over one large commit**.
- A commit must map to **exactly one type** (`feat`, `fix`, etc.).
- If a commit fits more than one type → **split it immediately**.
- Do NOT mix:
  - feature + fix
  - refactor + feature
  - docs + logic changes
- Each commit must be:
  - **independent**
  - **reviewable**
  - **revertible without side effects**

### Golden Rule

> If a commit can be split, it MUST be split.

---

## 3. Practical Guidelines

- Commit as soon as a **single unit of work is complete**.
- Avoid “batch commits” with multiple unrelated changes.
- Structure your workflow as:
  - write code → isolate change → commit
- If needed, use:
  - `git add -p` → stage partial changes
  - `git rebase -i` → split or fix commits before pushing

---

## 4. Examples (Atomic Commits)

### Basic

```bash
📗 docs: correct spelling of changelog
````

```bash
💻 feat(lang): add polish language
```

```bash
🛠️ fix(api): handle null response in user endpoint
```

---

### Breaking Change

```bash
💻 feat(api)!: send email when product is shipped
```

---

### Full Commit (with body and footer)

```text
🛠️ fix: prevent racing of requests

introduce request id and track latest request to ignore stale responses.
remove timeouts previously used as workaround.

Refs: #123
```

---

## 5. Anti-patterns (DO NOT DO)

### ❌ Multiple changes in one commit

```bash
💻 feat: add validation, fix bug and refactor service
```

### ❌ Mixed types

```bash
🛠️ fix: add new login endpoint
```

### ❌ Vague message

```bash
💻 feat: update stuff
```

---

## 6. Atomic Breakdown Example

Instead of:

```bash
💻 feat: implement order validation system
```

Do:

```bash
💻 feat: add order validation endpoint
💻 feat: create order validation service
🛠️ fix: handle null user in validation
⚙️ refactor: extract validation logic
🧪 test: add tests for order validation
📗 docs: document order validation endpoint
```

---

## Final Rule

> Better 10 small, clear commits than 1 large ambiguous commit.

```