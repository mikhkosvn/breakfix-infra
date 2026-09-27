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
