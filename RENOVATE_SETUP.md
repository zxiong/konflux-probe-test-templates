# Renovate Setup - Simple MintMaker Approach

This repository uses **the same setup as MintMaker** for automatic Tekton task updates with migrations.

## How It Works

Uses MintMaker's official Renovate image: `quay.io/konflux-ci/mintmaker-renovate-image`

This image already includes:
- ✅ `pipeline-migration-tool`
- ✅ `build-definitions` repository
- ✅ All dependencies (jq, yq, etc.)

**No custom scripts needed!**

## Setup (2 Files Only)

### 1. `renovate.json` - Configuration

Same config as MintMaker uses:
- Detects Konflux Tekton tasks
- Groups updates together
- Runs `pipeline-migration-tool` automatically

### 2. `.github/workflows/renovate.yml` - GitHub Actions

Runs Renovate using MintMaker's Docker image.

## Quick Start

### Commit and Push

```bash
cd /Users/jimmyxiong/projects/konflux/github/konflux-probe-test-templates

git add renovate.json .github/workflows/renovate.yml
git commit -m "Add Renovate with automatic migrations (MintMaker approach)"
git push
```

### Trigger Workflow

1. Go to **Actions** tab
2. Click **Renovate** workflow
3. Click **Run workflow**

**That's it!** ✨

## What You Get

When tasks update:
- ✅ Version updated automatically
- ✅ Parameters migrated automatically
- ✅ Single PR with all changes
- ✅ Links to MIGRATION.md

## Example PR

```
Title: Update Konflux task references

Changes:
✓ task-buildah: 0.1 → 0.2
✓ task-git-clone: 0.8 → 0.9

Automatic migrations applied:
- Added IMAGE_EXPIRES_AFTER parameter
- Updated git-clone authentication method
```

## Comparison to Complex Approach

**❌ Old Complex Way:**
- Custom migration script
- Install jq, yq, git locally
- Manage build-definitions repo
- 50+ lines of bash code

**✅ New Simple Way:**
- Use MintMaker's image
- 2 files only
- No custom scripts
- Zero dependencies to install

## How It's Exactly Like MintMaker

| Feature | MintMaker | Your Setup |
|---------|-----------|------------|
| Renovate image | `mintmaker-renovate-image` | ✅ Same |
| Migration tool | `pipeline-migration-tool` | ✅ Same |
| Migration scripts | `build-definitions` | ✅ Same |
| Configuration | `renovate.json` | ✅ Same |
| Custom scripts | None | ✅ None |

**The only difference:** You run it via GitHub Actions instead of Konflux running it.

## Troubleshooting

### "Image pull failed"

The image is public, no authentication needed. If it fails:

```yaml
# Try with docker.io mirror
RENOVATE_DOCKER_IMAGE: docker.io/konfluxci/mintmaker-renovate-image:latest
```

### "Command not allowed"

Check that `allowedCommands` in `renovate.json` includes:
```json
"allowedCommands": [
  "^pipeline-migration-tool migrate -f \"\\$RENOVATE_POST_UPGRADE_COMMAND_DATA_FILE\"$"
]
```

### Need help?

Open issue: https://github.com/konflux-ci/mintmaker/issues

## Resources

- MintMaker: https://github.com/konflux-ci/mintmaker
- Renovate image: https://quay.io/repository/konflux-ci/mintmaker-renovate-image
- Migration tool: https://github.com/konflux-ci/pipeline-migration-tool
