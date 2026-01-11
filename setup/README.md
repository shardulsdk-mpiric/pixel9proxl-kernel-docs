# Setup Instructions

This directory contains files needed to integrate this documentation repository with the Android kernel repo tool workflow.

## Quick Setup

To add this documentation to your Android kernel repository (using repo tool):

### Step 1: Copy Manifest File

Copy the manifest file to your repo's local_manifests directory:

```bash
# If local_manifests doesn't exist, create it
mkdir -p .repo/local_manifests

# Copy the manifest file
cp setup/pixel9proxl-docs.xml .repo/local_manifests/pixel9proxl-docs.xml
```

### Step 2: Sync Documentation

Run repo sync to download the documentation:

```bash
repo sync docs
```

This will clone the documentation repository into the `docs/` directory in your kernel source tree.

### Step 3: Access Documentation

After syncing, you can access the documentation at:

```
docs/README.md
```

## What This Does

The manifest file (`pixel9proxl-docs.xml`) tells the repo tool to:

1. Clone this documentation repository from GitHub
2. Place it in the `docs/` directory
3. Use the `main` branch
4. Sync it along with other kernel projects

## Manual Setup (Alternative)

If you prefer not to use repo tool, you can clone the documentation repository directly:

```bash
git clone https://github.com/shardulsdk-mpiric/pixel9proxl-kernel-docs.git docs
```

## Repository Information

- **GitHub**: https://github.com/shardulsdk-mpiric/pixel9proxl-kernel-docs
- **Branch**: `main`
- **Path**: `docs/` (when using repo tool)

## For Contributors

If you're contributing to the documentation, see the main [README.md](../README.md) and [MAINTENANCE.md](../MAINTENANCE.md) files.

## Troubleshooting

### Permission Denied Error

If you get permission errors, ensure:
1. You have SSH keys set up with GitHub (for SSH URLs)
2. Or use HTTPS URLs (already configured in manifest)

### Repo Sync Fails

If `repo sync docs` fails:
1. Check that `.repo/local_manifests/pixel9proxl-docs.xml` exists
2. Verify the manifest file syntax (it's valid XML)
3. Ensure you have network access to GitHub
4. Check that the repository exists and is accessible

### Docs Directory Not Created

If the `docs/` directory doesn't appear after `repo sync`:
1. Run `repo sync docs` explicitly (not just `repo sync`)
2. Check `.repo/project.list` to see if docs is listed
3. Check `.repo/local_manifests/` directory exists and contains the manifest file

