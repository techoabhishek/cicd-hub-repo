# Commit Message Format

Every commit message should follow this structure:

```
<type>(<scope>): <subject>

<body>

BREAKING CHANGE:
Related issue: #
```

## Commit Message Example

### Example 1
```shell
feat(auth): add JWT authentication
Implemented JWT authentication to improve security in the login flow.

BREAKING CHANGE: The authentication method has changed from session-based to JWT-based.

Related issue: #123
```

### Example 2
```shell
docs(README): update documentation for clarity
Added more details to the README to clarify the purpose of the repository.

BREAKING CHANGE: None

Related issue: None
```

## Available Commit Types

| Type     | Description |
|----------|-------------|
| feat     | A new feature |
| fix      | A bug fix |
| docs     | Documentation only changes |
| style    | Changes that do not affect the meaning of the code |
| refactor | Code change that neither fixes a bug nor adds a feature |
| perf     | Performance improvement |
| test     | Adding or correcting tests |
| chore    | Changes to the build process or auxiliary tools |

## How to Set Up the Commit Template in Your Repository

### 1. Create the Commit Template File

Create a file named `commit-template.txt` in the root of your repo or inside `.gitconfig/`.

Insert this content:

```shell
<type>(<scope>): <subject>

<body>

BREAKING CHANGE:
Related issue: #
```

### 2. Configure Git to Use This Template

Run this command:

```
git config commit.template .gitconfig/commit-template.txt
```

This will ensure that every time you make a commit using `git commit` ( without `-m` option ), the editor will open with the template, prompting you to fill in the necessary details.
