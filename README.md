# Pixel 9 Pro XL Kernel Build Documentation

## Overview

This documentation provides a comprehensive guide to building and customizing the kernel for the Pixel 9 Pro XL (codename: `caimito`). It covers the build process, GKI architecture, customization strategies, and practical guides for development and deployment.

## Quick Start

### Using Repo Tool (Recommended)

If you're using the Android kernel repo tool workflow, see [setup/README.md](setup/README.md) for instructions on adding this documentation to your repo sync.

**Quick setup:**
```bash
mkdir -p .repo/local_manifests
cp setup/pixel9proxl-docs.xml .repo/local_manifests/
repo sync docs
```

### Direct Clone

If you prefer to clone directly:
```bash
git clone https://github.com/shardulsdk-mpiric/pixel9proxl-kernel-docs.git docs
```

## Documentation Structure

### [Overview](overview.md)
High-level summary of the build process, device information, and key concepts. Start here if you're new to the project.

### [Analysis](analysis/README.md)
Deep-dive analyses of specific aspects of the build system. These documents provide detailed technical understanding of how the system works.

- [Analysis Plan](analysis/00-plan.md) - Planned areas for deep analysis
- [GKI Architecture](analysis/01-gki-architecture.md) - Understanding Generic Kernel Image architecture, mixed builds, and base kernel extension
- *More analyses coming...*

### [Guides](guides/README.md)
Step-by-step guides for common tasks and workflows.

- *Guides coming...*

### [Reference](reference/README.md)
Quick reference materials, configuration examples, and troubleshooting information.

- *Reference materials coming...*

## Quick Navigation

### I want to...

- **Understand the build process**: Start with [Overview](overview.md)
- **Learn about GKI architecture**: See [GKI Architecture Analysis](analysis/01-gki-architecture.md)
- **Build a custom kernel**: See [Overview](overview.md) → Building Custom Kernel section
- **Replace the GKI kernel**: See [GKI Architecture Analysis](analysis/01-gki-architecture.md) → How to Build Full Kernel Replacement
- **Find a specific configuration**: See [Reference](reference/README.md)

## Documentation Standards

This documentation follows these standards:

- **Format**: Markdown (`.md` files) for git-based maintenance
- **Structure**: Organized by topic in dedicated directories
- **Cross-references**: Use relative paths between documents
- **Code references**: Use code citation format `startLine:endLine:filepath`
- **Naming**: Use descriptive names with numbers for ordered content (e.g., `01-gki-architecture.md`)

## Maintenance

### Git-based Maintenance

All documentation is maintained in git. Follow these practices:

- **Commits**: Each logical change should be a separate commit with a clear message
- **Updates**: Update the relevant section when making changes to code or configuration
- **Cross-references**: Keep links between documents updated when files are moved or renamed

### Adding New Documentation

1. Choose the appropriate directory (`analysis/`, `guides/`, or `reference/`)
2. Use descriptive, kebab-case filenames (e.g., `02-build-configuration.md`)
3. Update the relevant README.md with a link to your new document
4. Add cross-references to related documents
5. Follow the existing formatting and structure patterns

### Publishing

This documentation can be published in multiple formats:

- **GitHub/GitLab**: View directly as Markdown
- **MkDocs**: Generate static website (recommended for website/wiki)
- **Sphinx**: Convert to RST for upstream contribution (Linux kernel docs format)

See [PUBLISHING.md](PUBLISHING.md) for details on publishing options.

## Related Documentation

- [Kleaf Documentation](../build/kernel/kleaf/docs/README.md) - Official Kleaf build system documentation
- [Android Kernel Build Guide](https://source.android.com/setup/build/building-kernels) - Official Android documentation

## Contributing

When contributing to this documentation:

1. Follow the existing structure and formatting
2. Keep language clear and accessible
3. Include code examples where helpful
4. Update cross-references when adding new content
5. Test all links before committing

## Status

**Current Status**: Documentation is under active development.

- ✅ Overview document
- ✅ GKI Architecture deep dive
- ⏳ Additional analyses in progress
- ⏳ Guides and reference materials in progress

Last updated: See git commit history

