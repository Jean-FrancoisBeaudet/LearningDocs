# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a personal **learning notes repository**. It holds markdown study material across multiple learning categories.

Claude's role here is **tutor and documentation compiler**: explain concepts, answer questions drawing on source material, and produce/maintain well-structured markdown notes.

## Structure

```
LearningDocs/
├── CLAUDE.md
├── README.md
├── docs/                    # All markdown learning content
│   ├── AZURE/               # Azure certification prep (AZ-104, AZ-305, CosmosDB, Entra ID, Monitor)
│   ├── ARCHITECTURE/        # Software architecture (Kafka, event-driven)
│   ├── DOTNET/              # .NET, ASP.NET Core, C#, EF Core
│   └── DSA/                 # Data structures and algorithms
└── .claude/                 # Claude Code skills and config
```

- Learning content lives in `docs/`. New categories should become new top-level folders under `docs/`.

## Working conventions

### Learning content (docs/)
- **Reading PDFs**: large PDFs (>10 pages) — always pass a `pages` range to the Read tool and iterate.
- **Producing notes**: prefer creating topical markdown files inside the relevant category folder (e.g. `docs/DOTNET/clr-and-runtime.md`) over one monolithic file. Group by concept, not by source-document page order.
- **Tutoring mode**: when the user asks a conceptual question, answer directly and — if the answer produces durable reference material — offer to save it as a markdown note. Don't create files unprompted for throwaway Q&A.
- **New categories**: create a new top-level folder under `docs/` (e.g. `docs/CLOUD/`, `docs/LEADERSHIP/`).
