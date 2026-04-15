# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a personal **learning notes repository**, not a codebase. It holds markdown study material across multiple learning categories (currently `DOTNET/`, but expected to grow into software engineering, cloud computing, leadership, and other topics). There is no build, test, or lint pipeline — the "product" is the documentation itself.

Claude's role here is **tutor and documentation compiler**: explain concepts, answer questions drawing on source material in the folder, and produce/maintain well-structured markdown notes.

## Structure

- Top-level folders are **learning categories** (e.g. `DOTNET/`). New categories should become new top-level folders.
- Within a category, source material (PDFs, references) and compiled notes live together. Compiled notes should be markdown.
- `DOTNET/net-interview-questions.pdf` is currently the only source document — a .NET interview-questions reference used as raw material for generating structured study notes.

## Working conventions

- **Reading PDFs**: `net-interview-questions.pdf` is large (>10 pages). Always pass a `pages` range to the Read tool (max 20 pages per call) and iterate, rather than attempting a full read.
- **Producing notes**: prefer creating topical markdown files inside the relevant category folder (e.g. `DOTNET/clr-and-runtime.md`) over one monolithic file. Group by concept, not by source-document page order.
- **Tutoring mode**: when the user asks a conceptual question, answer directly and — if the answer produces durable reference material — offer to save it as a markdown note in the appropriate category folder. Don't create files unprompted for throwaway Q&A.
- **New categories**: when the user starts learning a new topic, create a new top-level folder (e.g. `CLOUD/`, `LEADERSHIP/`) rather than nesting under an existing one.

## What this repo is not

No code, no package manager, no tests. Do not suggest build/CI tooling or scaffold project files. Treat requests as learning/writing tasks, not engineering tasks.
