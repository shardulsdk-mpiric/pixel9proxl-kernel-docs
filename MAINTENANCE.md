# Documentation Maintenance Guide

This guide explains how to maintain the Pixel 9 Pro XL kernel build documentation.

## Directory Structure

```
docs/
├── README.md              # Main index and navigation
├── overview.md            # High-level overview of build process
├── PUBLISHING.md          # Publishing strategies and options
├── MAINTENANCE.md         # This file
├── analysis/              # Deep-dive analyses
│   ├── README.md
│   ├── 00-plan.md        # Analysis plan
│   └── 01-gki-architecture.md
├── guides/                # Step-by-step guides
│   └── README.md
└── reference/             # Quick reference materials
    └── README.md
```

## File Naming Conventions

### Analysis Files
- Use numbered prefix: `00-plan.md`, `01-gki-architecture.md`, `02-*.md`
- Use kebab-case: lowercase with hyphens
- Use descriptive names: clearly indicate content

### Guide Files
- Use descriptive names without numbers (unless sequence matters)
- Use kebab-case
- Examples: `building-custom-kernel.md`, `flashing-kernel.md`

### Reference Files
- Use descriptive names
- Use kebab-case
- Examples: `configuration-reference.md`, `troubleshooting.md`

## Adding New Documentation

### 1. Choose the Right Location

- **Analysis**: Deep technical analysis → `analysis/`
- **Guide**: Step-by-step instructions → `guides/`
- **Reference**: Quick lookup → `reference/`

### 2. Create the File

- Use appropriate naming convention
- Start with frontmatter (title, navigation)
- Include navigation links

### 3. Update Index Files

- Update relevant `README.md` (analysis/README.md, guides/README.md, etc.)
- Add link to new document
- Update main `docs/README.md` if it's a major addition

### 4. Add Cross-References

- Link to related documents
- Use relative paths: `[text](../other-dir/file.md)`
- Link from related documents back to new document

## Writing Style

### Structure
- Start with overview/introduction
- Use clear headings (H2, H3, etc.)
- Break into logical sections
- Include code examples where helpful
- Add navigation section at top

### Code References
- Use code citation format: `startLine:endLine:filepath`
- Example: ```12:14:private/devices/google/caimito/BUILD.bazel```
- Include context around code snippets

### Links
- Use relative paths for internal links
- Use absolute URLs for external links
- Test all links before committing

## Git Workflow

### Commits
- Each logical change should be a separate commit
- Use clear commit messages:
  - `docs: Add guide for building custom kernel`
  - `docs: Update GKI architecture analysis with boot image details`
  - `docs: Fix broken link in overview`
- Follow conventional commits format if possible

### Branches
- Use feature branches for documentation updates
- Example: `docs/add-custom-kernel-guide`
- Merge to main after review

### Review
- Review documentation changes like code changes
- Check for:
  - Clarity and accuracy
  - Broken links
  - Formatting consistency
  - Navigation updates

## Keeping Documentation Updated

### When Code Changes
- Update relevant documentation
- Update code examples if they change
- Update configuration examples if needed

### When Understanding Changes
- Update analysis documents as understanding evolves
- Mark sections as "under review" if uncertain
- Add notes about iterative nature

### Regular Maintenance
- Review links periodically (broken link checkers)
- Update outdated information
- Add new sections as needed
- Remove obsolete content

## Version Control Best Practices

### What to Commit
- ✅ Markdown source files
- ✅ Configuration files (mkdocs.yml, etc.)
- ✅ Images/screenshots (if any)
- ✅ Navigation/index files

### What NOT to Commit
- ❌ Generated HTML/website files (if using MkDocs)
- ❌ Build artifacts
- ❌ Temporary files

### .gitignore
Consider adding to `.gitignore`:
```
# Generated documentation
docs/_build/
docs/site/
.mkdocs_cache/
```

## Navigation Maintenance

### Main README.md
- Keep table of contents updated
- Update "Quick Navigation" section
- Update status section

### Subdirectory READMEs
- Keep list of documents updated
- Keep navigation links updated
- Update status/progress indicators

### Cross-References
- When moving/renaming files, update all references
- Use `grep` to find all references:
  ```bash
  grep -r "old-filename" docs/
  ```
- Update references before moving files

## Publishing Workflow

### Local Testing
1. Review in text editor
2. Test Markdown rendering (GitHub preview, MkDocs serve)
3. Check all links
4. Verify navigation

### Publishing
1. Commit changes to git
2. Push to repository
3. If using automated publishing, changes deploy automatically
4. If manual publishing, follow PUBLISHING.md instructions

## Tools and Resources

### Markdown Editors
- VS Code with Markdown extensions
- Typora
- Mark Text
- Any text editor (for simple editing)

### Link Checking
- `grep -r "\[.*\](.*)" docs/` - Find all links
- Online link checkers
- MkDocs has link checking built-in

### Preview
- GitHub/GitLab: View in repository
- MkDocs: `mkdocs serve` for local preview
- VS Code: Markdown preview

## Questions or Issues

If you have questions about maintaining documentation:

1. Check this guide first
2. Check PUBLISHING.md for publishing questions
3. Review existing documentation for examples
4. Follow the existing patterns and structure

## Summary

**Key Principles**:
- Maintain in git (source of truth)
- Use clear structure and navigation
- Keep cross-references updated
- Follow naming conventions
- Review like code
- Update when code/understanding changes

**Workflow**:
1. Create/edit file in appropriate directory
2. Update navigation/index files
3. Add cross-references
4. Test links and preview
5. Commit with clear message
6. Review and merge

