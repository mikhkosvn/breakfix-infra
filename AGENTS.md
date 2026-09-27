# Agent rules — breakfix-infra

These rules apply to every agent that writes or edits a file in this repository.

This repository holds infrastructure code. The rules apply to every file that the repository
holds now, and to every file that an agent adds later.

The rules are not advice. An agent that breaks Rule 1 must repair the file before it reports
the work as complete.

## Rule 1 — Never write a comment

Do not write a comment in any file in this repository.

This rule has no exception for a resource that is hard to understand. It has no exception for
a workaround. It has no exception for a value that took a long time to make correct. Rule 3
names the only permitted comment syntax. That syntax is a directive for a tool, not a comment
for a reader.

## Rule 2 — What counts as a comment

The ban covers the comment syntax of every language in this repository.

| Language or file | Banned syntax |
|---|---|
| Terraform, in HashiCorp Configuration Language (HCL) | `#`, `//`, `/* */` |
| YAML Ain't Markup Language (YAML), and Kubernetes manifests | `#` |
| Dockerfile | `#` |
| Shell script | `#` |
| Makefile | `#` |
| Environment file, such as `.env.example` | `#` |
| Tom's Obvious Minimal Language (TOML), and `.ini` files | `#` |
| Markdown | `<!-- -->` |
| TypeScript and JavaScript, for Pulumi or for the Cloud Development Kit (CDK) | `//`, `/* */` |

The ban also covers these seven forms. Each one is a comment under a different name.

1. A file header that names the file, the author, the date, or the purpose.
2. A section banner, such as `# ---- networking ----`.
3. A comment above a resource block, a variable block, or a module block.
4. A marker such as `TODO` (work to do later), `FIXME` (a defect to repair later), `NOTE`,
   `HACK`, or `XXX`.
5. A resource that you disable with comment syntax instead of deleting it.
6. A comment in a test file, in a fixture, in a values file, or in a pipeline file.
7. A comment in a file that a template or a script of yours generates.

## Rule 3 — The only permitted comment syntax

Four forms use comment syntax, and a tool reads each one. The tool fails without the form, or
it reports a false alarm. These four forms are permitted. Nothing else is.

1. A shebang line at the top of an executable script, such as `#!/usr/bin/env bash`.
2. A Docker parser directive at the top of a Dockerfile, such as
   `# syntax=docker/dockerfile:1`.
3. A schema line that an editor reads, such as
   `# yaml-language-server: $schema=<url>`.
4. A suppression directive that a linter or a security scanner reads, such as
   `# tflint-ignore: <rule>`, `# tfsec:ignore:<rule>`, `# checkov:skip=<rule>`, or
   `# yamllint disable-line rule:<rule>`.

Write the directive alone. Never add prose to the same line. Never add a line of prose above
the directive.

Correct:

```hcl
# tfsec:ignore:aws-s3-enable-bucket-logging
```

Wrong:

```hcl
# tfsec:ignore:aws-s3-enable-bucket-logging because logs go to the audit account
```

## Rule 4 — Write the explanation in a place that is not the code

An explanation has value. The infrastructure file is the wrong place for it. Use one of these
seven places instead.

1. The name. Rename the resource, the variable, or the module until the name states the intent.
2. The `description` argument. A Terraform variable, output, and resource accept a description.
   That argument is a real field, not a comment. Use it.
3. A tag or a label. A cloud tag and a Kubernetes label both carry structured meaning.
4. A Kubernetes annotation. An annotation holds text that a reader and a tool can both read.
5. The commit message. Write the reason for the change there.
6. The pull request body. Write the design decision and the trade-off there.
7. A file in `docs/`. Write a long explanation there, and keep it out of the infrastructure file.

Report a risk or an open question to the user in your answer. Do not write it into the code.

## Rule 5 — Do not delete a comment that is already in the repository

Some files may already hold comments. Leave them.

Do not delete a comment that another author wrote. Ask the user first. One exception applies.
Delete a comment when you delete the resource that the comment describes.

## Rule 6 — Check your work before you report it

Read your own diff before you report the work as complete. Search the added lines for comment
syntax.

```bash
git diff -U0 | grep -E '^\+[^+]' | grep -E '(^|[[:space:]])#|//|/\*|<!--'
```

The command reports every added line that holds comment syntax. It also reports false matches,
such as a URL or a string. Read each match. Delete every match that is a comment.

## Rule 7 — Never write a test

Do not write a test in this repository. Do not create a test file. Do not add a test case to a
file that already exists.

This rule has no exception. A resource that is hard to understand does not earn a test. A defect
that you repaired does not earn a test. A request to "make sure it works" does not ask you for a
test.

## Rule 8 — What counts as a test

The ban covers every automated check, under every name.

| Form | Banned examples |
|---|---|
| Terraform test | `*.tftest.hcl`, and the `terraform test` command |
| Go test | `*_test.go`, and Terratest |
| Cloud Development Kit (CDK) or Pulumi test | `*.test.ts`, `*.spec.ts`, a Pytest file, a template assertion |
| Test function | `describe`, `it`, `test`, `run` blocks, `assert` blocks |
| Test double | a mock provider, a fake backend, or a fixture that exists for a test only |
| Test resource | a resource, a module, or a workspace that exists to prove another resource works |
| Runner configuration | a test stage in a pipeline file, `jest.config.*`, `vitest.config.*`, a `test` target in a Makefile |

## Rule 9 — A linter and a scanner are not tests

These tools stay, and Rule 3 permits their suppression directives.

- `terraform validate` and `terraform fmt`
- `terraform plan`
- `tflint`
- `tfsec` and `checkov`
- `yamllint`

Each one reads the code as it is. None of them builds a second copy of the system to check the
first one. That difference is what Rule 7 bans.

## Rule 10 — What to do instead of a test

Read the configuration and make it correct. Then report two things to the user.

1. What you changed.
2. What you did not verify.

Write the second part plainly. "I did not apply this against a real account" is a correct report.
Name each limit of what you observed. Never report a result that you did not see.

## Rule 11 — Check your work before you report it

Run this command before you report the work as complete.

```bash
git status --porcelain | grep -E '\.tftest\.hcl$|_test\.go$|\.(spec|test)\.(ts|js|py)$|__tests__'
```

The command must print nothing. One line of output means you broke Rule 7. Delete that file.
