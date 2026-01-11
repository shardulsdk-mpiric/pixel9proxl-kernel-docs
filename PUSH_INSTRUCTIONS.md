# Push Instructions

## Step 1: Create GitHub Repository

1. Go to: https://github.com/new
2. **Repository name**: `pixel9proxl-kernel-docs`
3. **Description**: `Documentation for Pixel 9 Pro XL (caimito) kernel build process`
4. **Visibility**: Public (recommended) or Private
5. **DO NOT** initialize with README, .gitignore, or license
6. Click "Create repository"

## Step 2: Push to GitHub

After creating the repository, run:

```bash
cd /tmp/pixel9proxl-kernel-docs-temp
git push -u origin main
```

## Step 3: Verify

Visit: https://github.com/shardulsdk-mpiric/pixel9proxl-kernel-docs
You should see all documentation files.

## Step 4: Test Repo Sync

In your 002_kernel directory:

```bash
cd /mnt/work_4gb/sources/android/pixel_9_pro_xl/002_kernel
repo sync docs
```

This should clone the docs repository into the `docs/` directory.
