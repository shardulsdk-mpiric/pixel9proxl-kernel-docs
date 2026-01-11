# Documentation Publishing Strategy

This document outlines strategies for publishing and maintaining the Pixel 9 Pro XL kernel build documentation.

## Git-based Maintenance

### Repository Structure

The documentation is maintained in the kernel repository alongside the source code:

```
002_kernel/
  docs/
    README.md              # Main index
    overview.md            # High-level overview
    PUBLISHING.md          # This file
    analysis/
      01-gki-architecture.md
      README.md
    guides/
      README.md
    reference/
      README.md
```

### Git Workflow

1. **Commits**: Each logical change should be a separate commit with a clear message
   - Example: `docs: Add full kernel replacement guide to GKI architecture analysis`
   
2. **Branches**: Use feature branches for documentation updates
   - Example: `docs/update-gki-analysis`
   
3. **Pull Requests**: Review documentation changes like code changes

4. **History**: Git history provides full change tracking and attribution

### Benefits of Git-based Maintenance

- **Version control**: Full history of all changes
- **Collaboration**: Multiple contributors can work on documentation
- **Review**: Changes can be reviewed before merging
- **Attribution**: Git tracks who made what changes
- **Rollback**: Easy to revert problematic changes

## Publishing Options

### Option 1: GitHub/GitLab (Markdown)

**Format**: Native Markdown rendering

**Pros**:
- No build step required
- Direct viewing in repository
- GitHub/GitLab automatically render Markdown
- Easy to navigate with links

**Cons**:
- Limited customization
- No search functionality (except repo search)
- Basic navigation

**Best for**: Quick reference, repository browsing

**Implementation**:
- Already supported by GitHub/GitLab
- Just commit Markdown files
- Links work automatically

### Option 2: MkDocs (Recommended for Website)

**Format**: Static website generated from Markdown

**Pros**:
- Professional-looking documentation sites
- Full-text search
- Better navigation (sidebar, table of contents)
- Easy to customize with themes
- Can be hosted on GitHub Pages, Netlify, etc.
- Still maintained in git (source is Markdown)

**Cons**:
- Requires build step (but can be automated)
- Need to maintain `mkdocs.yml` configuration

**Best for**: Published documentation website, wiki-style site

**Implementation**:
1. Install MkDocs: `pip install mkdocs mkdocs-material`
2. Create `mkdocs.yml` configuration file
3. Run `mkdocs build` to generate static site
4. Run `mkdocs serve` for local preview
5. Deploy to GitHub Pages, Netlify, etc.

**Example mkdocs.yml**:
```yaml
site_name: Pixel 9 Pro XL Kernel Build Documentation
site_description: Comprehensive guide to building and customizing the Pixel 9 Pro XL kernel

theme:
  name: material
  palette:
    - scheme: default
      primary: blue
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode

nav:
  - Home: README.md
  - Overview: overview.md
  - Analysis:
    - Analysis Index: analysis/README.md
    - GKI Architecture: analysis/01-gki-architecture.md
  - Guides:
    - Guides Index: guides/README.md
  - Reference:
    - Reference Index: reference/README.md

markdown_extensions:
  - codehilite
  - tables
  - toc:
      permalink: true
```

### Option 3: Sphinx (For Upstream Contribution)

**Format**: RST-based documentation (Linux kernel standard)

**Pros**:
- Standard format for Linux kernel documentation
- Can be integrated into upstream kernel docs
- Professional output (PDF, HTML, etc.)
- Full-featured documentation system

**Cons**:
- Requires converting Markdown to RST
- More complex setup
- Overkill for standalone documentation

**Best for**: Upstream contribution to Linux kernel documentation

**Implementation**:
1. Convert Markdown to RST (tools available)
2. Set up Sphinx configuration
3. Build documentation with Sphinx
4. Can integrate into `Documentation/` directory of kernel

### Option 4: Hybrid Approach (Recommended)

**Recommended Strategy**:

1. **Primary**: Maintain in git as Markdown
   - Source of truth
   - Easy to edit and review
   - Version controlled

2. **Website**: Generate with MkDocs
   - Automated build on commit (GitHub Actions, CI/CD)
   - Deploy to website
   - Better user experience

3. **Upstream**: Convert to RST when ready
   - Keep Markdown as source
   - Convert for upstream submission
   - Maintain both formats if needed

## Implementation Plan

### Phase 1: Git-based Maintenance (Current)

- ✅ Organize documentation in `docs/` directory
- ✅ Use Markdown format
- ✅ Maintain in git repository
- ✅ Create proper structure and navigation

### Phase 2: Website Publishing (Recommended Next Step)

1. Create `mkdocs.yml` configuration
2. Set up automated build (GitHub Actions)
3. Deploy to GitHub Pages or personal website
4. Add search functionality
5. Improve navigation

### Phase 3: Upstream Contribution (Future)

1. Evaluate documentation quality
2. Identify upstreamable content
3. Convert to RST format
4. Submit to appropriate upstream projects
   - Android kernel documentation
   - Kleaf documentation
   - Device-specific documentation

## Recommended Setup: MkDocs + GitHub Pages

### Quick Start

1. **Install MkDocs**:
   ```bash
   pip install mkdocs mkdocs-material
   ```

2. **Create mkdocs.yml** (see example above)

3. **Test locally**:
   ```bash
   mkdocs serve
   ```

4. **Build**:
   ```bash
   mkdocs build
   ```

5. **Deploy to GitHub Pages**:
   ```bash
   mkdocs gh-deploy
   ```

### GitHub Actions Automation

Create `.github/workflows/docs.yml`:

```yaml
name: Deploy Documentation

on:
  push:
    branches:
      - main
    paths:
      - 'docs/**'
      - 'mkdocs.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.x'
      - run: pip install mkdocs mkdocs-material
      - run: mkdocs gh-deploy --force
```

## Maintaining Both Formats (Markdown + Website)

1. **Source**: Markdown files in `docs/` (version controlled)
2. **Build**: Automated via CI/CD
3. **Output**: Static site (generated, can be in `.gitignore`)
4. **Publishing**: Automated deployment on commits

## License and Attribution

When publishing:

- Maintain original license (if any)
- Attribute contributors (git history provides this)
- Link back to source repository
- Consider CC-BY-SA for documentation (allows sharing with attribution)

## Conclusion

**Recommended approach**:
1. Maintain documentation in git as Markdown (current)
2. Set up MkDocs for website generation
3. Automate deployment with GitHub Actions
4. Consider upstream contribution when documentation matures

This provides:
- ✅ Easy maintenance (Markdown in git)
- ✅ Professional website (MkDocs)
- ✅ Automated publishing (CI/CD)
- ✅ Version control (git)
- ✅ Future upstream possibility (can convert to RST)

