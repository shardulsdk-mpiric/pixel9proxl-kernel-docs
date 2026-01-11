# Documentation Structure

This document explains the structure and organization of the Pixel 9 Pro XL kernel build documentation.

## Overview

The documentation is organized into a clear hierarchy:

```
docs/
├── README.md              # Main index - START HERE
├── overview.md            # High-level overview
├── PUBLISHING.md          # Publishing strategies
├── MAINTENANCE.md         # Maintenance guide
├── STRUCTURE.md           # This file
├── analysis/              # Deep technical analyses
│   ├── README.md
│   ├── 00-plan.md
│   └── 01-gki-architecture.md
├── guides/                # Step-by-step guides
│   └── README.md
└── reference/             # Quick reference
    └── README.md
```

## Directory Purposes

### Root Level (`docs/`)

- **README.md**: Main entry point with table of contents and navigation
- **overview.md**: High-level overview of the build process
- **PUBLISHING.md**: Publishing strategies and options
- **MAINTENANCE.md**: How to maintain the documentation
- **STRUCTURE.md**: This file - explains the structure

### analysis/ (Deep Technical Analysis)

Contains detailed technical analyses of specific aspects of the build system.

- Focus: Understanding how things work
- Audience: Developers wanting deep technical understanding
- Format: Detailed explanations with code references
- Naming: Numbered prefix (`00-`, `01-`, etc.) for ordered content

### guides/ (Step-by-Step Guides)

Contains practical, step-by-step guides for common tasks.

- Focus: How to do something
- Audience: Developers wanting to accomplish specific tasks
- Format: Step-by-step instructions with examples
- Naming: Descriptive names without numbers (unless sequence matters)

### reference/ (Quick Reference)

Contains quick reference materials, configuration examples, troubleshooting.

- Focus: Quick lookup
- Audience: Developers needing quick information
- Format: Concise reference materials
- Naming: Descriptive names

## Navigation

Each document includes navigation links at the top:
- Links to parent directory README
- Links to related documents
- Links to main index

## File Organization Principles

1. **Logical Grouping**: Related content grouped together
2. **Clear Hierarchy**: Main index → Categories → Individual documents
3. **Easy Navigation**: Multiple navigation paths (index, cross-references)
4. **Scalability**: Structure supports adding new content easily

## Finding Information

### By Topic
- **Build process overview**: `overview.md`
- **GKI architecture**: `analysis/01-gki-architecture.md`
- **Step-by-step guides**: `guides/`
- **Quick reference**: `reference/`

### By Goal
- **I want to understand**: Start with `overview.md` or `analysis/`
- **I want to do something**: See `guides/`
- **I need quick info**: See `reference/`

### By Navigation
- Start at `README.md` for table of contents
- Use navigation links in each document
- Follow cross-references between documents

## Adding New Content

1. Determine category (analysis/guides/reference)
2. Create file with appropriate naming
3. Add to category README
4. Add to main README if major addition
5. Add cross-references to related documents

See [MAINTENANCE.md](MAINTENANCE.md) for detailed instructions.
