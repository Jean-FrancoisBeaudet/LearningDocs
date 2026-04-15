---
description: Answer questions from the DOTNET topic markdown files and save to a new markdown file
argument-hint: <section> <question-range> (e.g. "asp.net core 1-5", "ef core 10", "c# 3-7,12")
allowed-tools: Read, Write, Glob, Grep, WebSearch, WebFetch, Bash, Skill
---

You are answering questions from the curated DOTNET interview-question markdown files for the user's study notes.

## Source files

Each section has its own markdown file listing the numbered questions:

- **ASP.NET Core** → `DOTNET/ASPNETCORE/aspnetcore-interview-questions.md` (60 questions)
- **EF Core** → `DOTNET/EFCORE/efcore-interview-questions.md` (40 questions; grouped under Junior / Mid-Senior headings, but numbering is continuous 1–40)
- **C#** → `DOTNET/C#/csharp-interview-questions.md` (50 questions)

## Arguments

User input: `$ARGUMENTS`

Parse it into:
- **section**: one of `Asp.Net Core`, `EF Core`, `C#` (normalize case-insensitively)
- **question numbers**: a range like `1-5`, a single number `10`, or a comma-separated list `3-7,12`

If arguments are missing or ambiguous, ask the user before proceeding.

## Procedure

0. **Invoke the `dotnet-senior` skill first** via the Skill tool (`skill: "dotnet-senior"`). All answers in this command must be produced under that skill's senior-.NET-expert guidance — depth, accuracy, and framing should match a senior engineer's explanation, not a surface-level summary.

1. **Load the section file.** Use the Read tool to open the appropriate markdown file from the list above. These files are small — a single Read call is sufficient.

2. **Extract the requested questions** by their numbers. Capture each question's wording faithfully from the file.

3. **Answer each question yourself**, targeting the **latest stable versions**: **.NET 10** and **C# 14**. If the question reflects older framework behavior, note what has changed and give the modern answer.

4. **Use web search when needed** to confirm .NET 10 / C# 14 specifics, new APIs, deprecations, or recent best practices. Prefer official Microsoft Learn / .NET blog / C# language spec sources. Cite the URL inline when a fact comes from the web.

5. **Write a new markdown file** in the same folder as the source file (so answers sit alongside their question list). Filename format:

   ```
   <section-folder>/<section-slug>-q<range>-<YYYYMMDD>.md
   ```

   Examples:
   - `DOTNET/ASPNETCORE/aspnet-core-q1-5-20260414.md`
   - `DOTNET/EFCORE/ef-core-q10-20260414.md`
   - `DOTNET/C#/csharp-q3-7,12-20260414.md`

   If the same file already exists, append `-v2`, `-v3`, etc. Never overwrite an existing answer file.

6. **File structure:**

   ```markdown
   # <Section> — Questions <range>

   _Source: `<source-file-path>`. Answers target .NET 10 / C# 14._

   ## Q<n>. <question text>

   <answer — clear, concept-first, with short code examples in ```csharp fences where useful>

   **Modern note (.NET 10 / C# 14):** <only if the modern answer differs materially from older framework behavior>

   **Sources:** <bullet list of URLs if web search was used for this question; omit otherwise>

   ---
   ```

   Repeat per question. Keep answers focused and senior-level — explain the *why*, not just the *what*. Prefer accurate depth over length.

7. **After writing**, report the created file path and a one-line summary of what was covered. Do not echo the full answers back into chat.

## Guardrails

- Don't fabricate .NET 10 / C# 14 features. If unsure whether something landed in the GA release, web-search to confirm or mark it as "preview / proposed".
- Don't create the file until you have actually answered every requested question.
- One new file per invocation — never mutate prior answer files or the source question-list files.
