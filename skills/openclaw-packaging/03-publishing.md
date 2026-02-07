# Publishing Skills

## Overview

This lesson covers the complete workflow for publishing OpenClaw skills: validating format, generating distribution packages, publishing to the OpenClaw registry, git-based distribution, versioning workflows, updating published skills, and deprecating outdated skills.

## Pre-Publication Validation

Before publishing, validate your skill package:

```typescript
import { readdir, readFile } from 'fs/promises';
import { join } from 'path';
import yaml from 'yaml';

interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
}

async function validateSkill(skillPath: string): Promise<ValidationResult> {
  const result: ValidationResult = {
    valid: true,
    errors: [],
    warnings: []
  };
  
  // Check SKILL.md exists
  const skillMdPath = join(skillPath, 'SKILL.md');
  try {
    const content = await readFile(skillMdPath, 'utf-8');
    
    // Parse frontmatter
    const frontmatterMatch = content.match(/^---\n([\s\S]*?)\n---/);
    if (!frontmatterMatch) {
      result.errors.push('SKILL.md missing YAML frontmatter');
      result.valid = false;
    } else {
      const frontmatter = yaml.parse(frontmatterMatch[1]);
      
      // Validate required fields
      if (!frontmatter.name) {
        result.errors.push('Missing required field: name');
        result.valid = false;
      } else {
        // Check name matches directory
        const dirName = skillPath.split('/').pop();
        if (frontmatter.name !== dirName) {
          result.errors.push(
            `Name mismatch: directory "${dirName}" vs ` +
            `frontmatter "${frontmatter.name}"`
          );
          result.valid = false;
        }
      }
      
      if (!frontmatter.version) {
        result.errors.push('Missing required field: version');
        result.valid = false;
      } else if (!/^\d+\.\d+\.\d+/.test(frontmatter.version)) {
        result.errors.push(
          `Invalid version format: ${frontmatter.version} ` +
          `(expected semver: X.Y.Z)`
        );
        result.valid = false;
      }
      
      if (!frontmatter.description) {
        result.errors.push('Missing required field: description');
        result.valid = false;
      }
      
      if (!frontmatter.trigger_phrases || 
          frontmatter.trigger_phrases.length < 3) {
        result.warnings.push(
          'Recommended: At least 3 trigger phrases for better discovery'
        );
      }
    }
    
    // Check for non-ASCII characters
    const nonAscii = content.match(/[^\x00-\x7F]/g);
    if (nonAscii) {
      result.errors.push(
        `Non-ASCII characters found: ${nonAscii.slice(0, 5).join(', ')}...`
      );
      result.valid = false;
    }
    
  } catch (error) {
    result.errors.push('SKILL.md not found or not readable');
    result.valid = false;
  }
  
  // Check for lesson files
  const files = await readdir(skillPath);
  const lessonFiles = files.filter(f => /^\d{2}-.*\.md$/.test(f));
  
  if (lessonFiles.length === 0) {
    result.warnings.push('No lesson files found (01-*.md, 02-*.md, etc.)');
  }
  
  // Validate lesson file ordering
  const numbers = lessonFiles.map(f => parseInt(f.slice(0, 2)));
  for (let i = 0; i < numbers.length - 1; i++) {
    if (numbers[i + 1] !== numbers[i] + 1) {
      result.warnings.push(
        `Lesson file gap: ${numbers[i]} -> ${numbers[i + 1]}`
      );
    }
  }
  
  // Check for manifest
  const hasManifest = files.includes('manifest.yaml') || 
                      files.includes('package.json');
  if (!hasManifest) {
    result.warnings.push(
      'No manifest.yaml or package.json found (recommended)'
    );
  }
  
  return result;
}

// Usage
const validation = await validateSkill('./skills/base-token-swaps');
if (!validation.valid) {
  console.error('Validation failed:');
  validation.errors.forEach(e => console.error(`  - ${e}`));
  process.exit(1);
}

if (validation.warnings.length > 0) {
  console.warn('Warnings:');
  validation.warnings.forEach(w => console.warn(`  - ${w}`));
}
```

## Package Generation

Create a tarball for distribution:

```typescript
import { createWriteStream } from 'fs';
import { readdir, readFile } from 'fs/promises';
import { join } from 'path';
import tar from 'tar';
import yaml from 'yaml';

async function packageSkill(
  skillPath: string,
  outputDir: string
): Promise<string> {
  // Read manifest
  const manifestPath = join(skillPath, 'manifest.yaml');
  const manifestContent = await readFile(manifestPath, 'utf-8');
  const manifest = yaml.parse(manifestContent);
  
  // Determine files to include
  let files: string[];
  if (manifest.files) {
    files = manifest.files;
  } else {
    // Default: all .md files and contracts/
    const allFiles = await readdir(skillPath);
    files = allFiles.filter(f => 
      f.endsWith('.md') || 
      f === 'manifest.yaml' ||
      f === 'package.json' ||
      f === 'contracts'
    );
  }
  
  // Create tarball
  const tarballName = `${manifest.name}-${manifest.version}.tgz`;
  const tarballPath = join(outputDir, tarballName);
  
  await tar.create(
    {
      gzip: true,
      file: tarballPath,
      cwd: skillPath
    },
    files
  );
  
  console.log(`Created package: ${tarballPath}`);
  return tarballPath;
}

// Usage
const tarball = await packageSkill(
  './skills/base-token-swaps',
  './dist'
);
```

## OpenClaw Registry Publishing

Publish to the OpenClaw registry for centralized discovery:

```typescript
import { createReadStream } from 'fs';
import { readFile } from 'fs/promises';
import FormData from 'form-data';
import fetch from 'node-fetch';

const REGISTRY_URL = 'https://registry.openclaw.ai';

interface PublishOptions {
  tarballPath: string;
  apiKey: string;
  dryRun?: boolean;
}

async function publishToRegistry(
  options: PublishOptions
): Promise<void> {
  // Create form data with tarball
  const form = new FormData();
  form.append('package', createReadStream(options.tarballPath));
  
  // Add metadata
  form.append('dry_run', options.dryRun ? 'true' : 'false');
  
  // Upload to registry
  const response = await fetch(`${REGISTRY_URL}/api/publish`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${options.apiKey}`,
      ...form.getHeaders()
    },
    body: form
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(
      `Publishing failed: ${error.message || response.statusText}`
    );
  }
  
  const result = await response.json();
  console.log('Published successfully:');
  console.log(`  Name: ${result.name}`);
  console.log(`  Version: ${result.version}`);
  console.log(`  URL: ${result.url}`);
}

// Usage
await publishToRegistry({
  tarballPath: './dist/base-token-swaps-1.2.0.tgz',
  apiKey: process.env.OPENCLAW_API_KEY!,
  dryRun: false
});
```

**Registry authentication:**

Get an API key from the OpenClaw registry:

```bash
# Login to registry
openclaw login

# Generate API key
openclaw token create --name "CI/CD Publishing"

# Store in environment
export OPENCLAW_API_KEY="oc_..."
```

## Git-Based Distribution

Distribute skills via Git repositories:

```bash
# Initialize skill repository
cd skills/base-token-swaps
git init
git add .
git commit -m "Initial commit: v1.0.0"

# Tag version
git tag -a v1.0.0 -m "Release version 1.0.0"

# Push to remote
git remote add origin https://github.com/openclaw/base-token-swaps.git
git push origin main --tags
```

**Install from Git:**

```bash
# Install specific version
openclaw install github:openclaw/base-token-swaps#v1.0.0

# Install latest
openclaw install github:openclaw/base-token-swaps

# Install from branch
openclaw install github:openclaw/base-token-swaps#develop
```

**Git-based skill manifest:**

```yaml
name: base-token-swaps
version: 1.2.0
repository:
  type: git
  url: https://github.com/openclaw/base-token-swaps.git
  directory: "."  # Root of repo

# Or monorepo structure
repository:
  type: git
  url: https://github.com/openclaw/baised-skills.git
  directory: "skills/base-token-swaps"  # Subdirectory
```

## Version Workflow

Follow this workflow for version management:

**1. Development phase:**
```bash
# Create feature branch
git checkout -b feature/add-multi-hop-swaps

# Make changes
echo "# Multi-Hop Swaps" > 04-multi-hop-swaps.md
# ... write content ...

# Commit
git add 04-multi-hop-swaps.md
git commit -m "Add multi-hop swaps lesson"
```

**2. Version bump:**
```bash
# Determine version bump type
# - Major: Breaking changes
# - Minor: New features (new lesson files)
# - Patch: Bug fixes, typos

# Update version in SKILL.md
# Before: version: 1.1.0
# After:  version: 1.2.0

# Update version in manifest.yaml
sed -i 's/version: 1.1.0/version: 1.2.0/' manifest.yaml

# Commit version bump
git add SKILL.md manifest.yaml
git commit -m "Bump version to 1.2.0"
```

**3. Create release:**
```bash
# Merge to main
git checkout main
git merge feature/add-multi-hop-swaps

# Create annotated tag
git tag -a v1.2.0 -m "Release v1.2.0: Add multi-hop swaps"

# Push with tags
git push origin main --tags
```

**4. Publish:**
```bash
# Validate
openclaw validate ./skills/base-token-swaps

# Package
openclaw package ./skills/base-token-swaps

# Publish to registry
openclaw publish ./dist/base-token-swaps-1.2.0.tgz
```

**Automated workflow (GitHub Actions):**

```yaml
# .github/workflows/publish.yml
name: Publish Skill

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install OpenClaw CLI
        run: npm install -g @openclaw/cli
      
      - name: Validate skill
        run: openclaw validate .
      
      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Package skill
        run: openclaw package . --output ./dist
      
      - name: Publish to registry
        env:
          OPENCLAW_API_KEY: ${{ secrets.OPENCLAW_API_KEY }}
        run: |
          openclaw publish ./dist/*-${{ steps.version.outputs.VERSION }}.tgz
      
      - name: Create GitHub release
        uses: softprops/action-gh-release@v1
        with:
          files: ./dist/*.tgz
          generate_release_notes: true
```

## Updating Published Skills

Update existing skills with new versions:

```typescript
import { readFile, writeFile } from 'fs/promises';
import yaml from 'yaml';
import { inc } from 'semver';

interface UpdateOptions {
  skillPath: string;
  bumpType: 'major' | 'minor' | 'patch';
  changelog: string;
}

async function updateSkillVersion(
  options: UpdateOptions
): Promise<void> {
  // Read current manifest
  const manifestPath = `${options.skillPath}/manifest.yaml`;
  const manifestContent = await readFile(manifestPath, 'utf-8');
  const manifest = yaml.parse(manifestContent);
  
  // Bump version
  const oldVersion = manifest.version;
  const newVersion = inc(oldVersion, options.bumpType);
  manifest.version = newVersion;
  
  // Write updated manifest
  await writeFile(manifestPath, yaml.stringify(manifest));
  
  // Update SKILL.md frontmatter
  const skillMdPath = `${options.skillPath}/SKILL.md`;
  const skillContent = await readFile(skillMdPath, 'utf-8');
  const updatedSkill = skillContent.replace(
    /version: [\d.]+/,
    `version: ${newVersion}`
  );
  await writeFile(skillMdPath, updatedSkill);
  
  // Update CHANGELOG.md
  const changelogPath = `${options.skillPath}/CHANGELOG.md`;
  let changelog = '';
  try {
    changelog = await readFile(changelogPath, 'utf-8');
  } catch {
    changelog = '# Changelog\n\n';
  }
  
  const date = new Date().toISOString().split('T')[0];
  const newEntry = `## [${newVersion}] - ${date}\n\n${options.changelog}\n\n`;
  const updatedChangelog = changelog.replace(
    '# Changelog\n\n',
    `# Changelog\n\n${newEntry}`
  );
  await writeFile(changelogPath, updatedChangelog);
  
  console.log(`Updated version: ${oldVersion} -> ${newVersion}`);
}

// Usage
await updateSkillVersion({
  skillPath: './skills/base-token-swaps',
  bumpType: 'minor',
  changelog: '### Added\n- Multi-hop swap support\n- Gas optimization examples'
});
```

## Skill Deprecation

Mark skills as deprecated when superseded or no longer maintained:

```yaml
# manifest.yaml
name: old-token-swaps
version: 1.5.0
description: "[DEPRECATED] Use base-token-swaps instead"

deprecated: true
deprecation:
  reason: Superseded by improved implementation
  replacement: base-token-swaps
  date: "2024-03-15"
  migration_guide: https://docs.openclaw.ai/migration/old-to-new

warning: |
  This skill is deprecated and will be removed in a future version.
  Please migrate to base-token-swaps v2.0.0 or later.
```

**Deprecation notice in SKILL.md:**

```markdown
---
name: old-token-swaps
description: "[DEPRECATED] Legacy token swap implementation"
version: 1.5.0
---

# [DEPRECATED] Old Token Swaps

**DEPRECATION NOTICE:** This skill is deprecated as of March 15, 2024.
Please use the new `base-token-swaps` skill instead.

Migration guide: https://docs.openclaw.ai/migration/old-to-new

## Guidelines

This skill is no longer maintained...
```

**Deprecation workflow:**

```bash
# 1. Mark as deprecated
echo "deprecated: true" >> manifest.yaml
git commit -am "Mark skill as deprecated"

# 2. Publish deprecation version
git tag v1.5.0
git push --tags
openclaw publish

# 3. Wait grace period (e.g., 6 months)

# 4. Archive or remove
openclaw unpublish old-token-swaps
# or
openclaw archive old-token-swaps
```

## Publishing Checklist

Before publishing, verify:

```markdown
## Pre-Publish Checklist

- [ ] All files use ASCII-only characters
- [ ] SKILL.md name matches directory name
- [ ] Version follows semver (X.Y.Z)
- [ ] At least 3 trigger phrases
- [ ] All lesson files numbered correctly (01-, 02-, etc.)
- [ ] All code examples are syntactically correct
- [ ] All contract addresses are real and checksummed
- [ ] Base chain ID is 8453
- [ ] Cross-references link to existing skills
- [ ] No placeholders ("..." or "// more code here")
- [ ] Manifest includes required fields
- [ ] Dependencies use version ranges (^1.0.0)
- [ ] Tags include primary category
- [ ] CHANGELOG.md updated (if exists)
- [ ] Tests pass (if exists)
- [ ] Documentation complete

## Validation Commands

```bash
openclaw validate .
openclaw lint .
openclaw test .
```

## Publish Commands

```bash
# Dry run
openclaw publish --dry-run

# Actual publish
openclaw publish

# Or with explicit version
openclaw publish --tag v1.2.0
```
```

## Common Patterns

**Monorepo publishing:**

```bash
# Publish multiple skills from monorepo
for skill in skills/*/; do
  echo "Publishing ${skill}..."
  cd "$skill"
  openclaw validate .
  openclaw publish .
  cd -
done
```

**Conditional publishing:**

```bash
# Only publish if version tag matches manifest
MANIFEST_VERSION=$(grep 'version:' manifest.yaml | awk '{print $2}')
GIT_TAG=${GITHUB_REF#refs/tags/v}

if [ "$MANIFEST_VERSION" = "$GIT_TAG" ]; then
  openclaw publish .
else
  echo "Version mismatch: manifest=$MANIFEST_VERSION, tag=$GIT_TAG"
  exit 1
fi
```

**Multi-registry publishing:**

```bash
# Publish to multiple registries
openclaw publish --registry https://registry.openclaw.ai
openclaw publish --registry https://backup-registry.example.com
npm publish  # Also publish as npm package
```

## Gotchas

**Version immutability:** Once published, a version cannot be changed:

```bash
# This will fail if v1.2.0 already exists
openclaw publish base-token-swaps-1.2.0.tgz
# Error: Version 1.2.0 already published

# Solution: Increment version to 1.2.1
```

**Tag format:** Git tags should match manifest version:

```bash
# Manifest: version: 1.2.0
git tag v1.2.0    # Correct (with 'v' prefix)
git tag 1.2.0     # Also acceptable
git tag 1.2       # Wrong (missing patch version)
```

**Publish order:** Publish dependencies before dependent skills:

```bash
# Wrong order (will fail)
openclaw publish base-token-swaps  # Depends on token-standards
openclaw publish token-standards

# Correct order
openclaw publish token-standards    # Publish dependency first
openclaw publish base-token-swaps   # Then dependent skill
```

**Registry caching:** Registry may cache old versions for up to 5 minutes:

```bash
# Force cache invalidation
openclaw publish --force-update

# Or wait for cache to expire
sleep 300  # 5 minutes
```

**Tarball size:** Keep packages under 10MB:

```bash
# Check package size
ls -lh dist/base-token-swaps-1.2.0.tgz

# If too large, exclude unnecessary files
echo "node_modules/" >> .openclawignore
echo "test/" >> .openclawignore
echo "*.log" >> .openclawignore
```}