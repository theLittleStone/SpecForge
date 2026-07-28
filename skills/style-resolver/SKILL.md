---
name: style-resolver
description: 根据编程语言和框架生成或补全项目风格规范，并维护统一的 style_guide.md 文件
compatibility: opencode
---

## What I do
- Detect the project's language and framework
- Generate `style_guide.md` based on existing code or community conventions
- Update existing guides with incremental changes

## When to use me
Use me when `./style_guide.md` doesn't exist, or when asked to regenerate it.

## How I work
1. Scan source file extensions with `glob` to identify the language. If unclear, ask the user.
2. If code exists, read 3 files to detect indentation, naming, and quote style — use those.
3. If no code exists, use community conventions for that language.
4. Before writing, read `style-guide-example.md` in this skill's
   directory for the exact format. Follow that structure, replacing values with
   the project's actual style data. Annotate where each rule came from.
5. If `style_guide.md` already exists, use `edit` for changes — don't overwrite everything.

If you don't know the conventions for a language, ask the user or search — don't guess.
