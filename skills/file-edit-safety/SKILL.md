---
name: file-edit-safety
description: Preserve existing file bytes, encodings, BOM state, line endings, and unrelated edits when Codex modifies files. Use for any task that edits existing files, especially with PowerShell, Set-Content, redirection, generated rewrites, mojibake, legacy encodings, CRLF/LF changes, non-UTF-8 bytes, or files that already contain garbled text.
---

# File Edit Safety

Use this skill before modifying existing files.

## Core Rules

- Treat existing bytes as user data. Preserve encoding, BOM state, line endings, final newline style, and unrelated formatting unless the user explicitly asks to migrate them.
- Prefer patch-style edits that touch only the intended lines.
- Avoid whole-file rewrites for small changes.
- Do not use PowerShell `Set-Content`, shell redirection, pretty-printers, or formatters on existing files unless preserving encoding and line endings is intentional and verified.
- Be extra careful with files containing mojibake, mixed encodings, non-ASCII text, generated content, certificates, secrets, lockfiles, CSV exports, YAML manifests, and Markdown skills.

## Required Workflow

1. Inspect the file and current git status before editing.
2. Choose the narrowest edit method available.
3. If normal patch tools cannot match because of encoding or line-ending issues, use a byte-preserving insertion or replacement strategy.
4. After editing, run `git diff --stat` and inspect the exact `git diff`.
5. If a small content change produces a large diff, restore only your change and redo it byte-preservingly.
6. Mention any intentional encoding, BOM, or line-ending changes explicitly in the final response.

## PowerShell Notes

- `Set-Content` can silently alter encoding, add or remove BOM, normalize line endings, or rewrite the whole file.
- `[System.IO.File]::WriteAllText()` can also introduce encoding/BOM differences depending on overload and runtime.
- For existing files with fragile encoding, prefer `[System.IO.File]::ReadAllBytes()` and `[System.IO.File]::WriteAllBytes()` with targeted byte insertion/replacement.
- If writing a brand-new file, normal text writes are acceptable; this rule is mainly about preserving existing files.

## Validation Standard

A safe edit has a diff proportional to the requested change. For example, adding one paragraph should not rewrite the rest of the file, change the first line to include a BOM, or alter unrelated line endings.
