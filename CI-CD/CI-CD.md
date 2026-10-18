# 🚀 Universal CI/CD Setup & npm Automated Publishing Guide

A comprehensive, battle-tested reference guide for setting up Continuous Integration (CI) and Continuous Deployment (CD) pipelines using GitHub Actions. This document covers both a **General CI/CD Workflow** for any project and a **Dedicated npm Library Publishing Pipeline** for Node.js/TypeScript packages.

---

## 📑 Table of Contents
1. [Part 1: General CI/CD Setup Process](#part-1-general-cicd-setup-process)
2. [Part 2: Complete npm Publishing Pipeline (TypeScript/Node.js)](#part-2-complete-npm-publishing-pipeline-typescriptnodejs)
3. [Part 3: Managing Secrets & Security](#part-3-managing-secrets--security)
4. [Part 4: Release Workflow & Best Practices](#part-4-release-workflow--best-practices)

---

# Part 1: General CI/CD Setup Process

Continuous Integration (CI) ensures code quality through automated testing and builds, while Continuous Deployment (CD) delivers verified code automatically to production environments (e.g., AWS, Vercel, Docker Hub, npm).

### The Standard CI/CD Lifecycle

```
[ Local Dev ] ──> [ Git Push ] ──> [ CI: Test & Build ] ──> [ CD: Deployment ]
```

### Core Steps for Any CI/CD Setup

1. **Define Triggers (`on`)**:
   - Determine when workflows run (e.g., on `push` to `main`, on `pull_request`, or when a `release` is created).

2. **Select the Environment (`runs-on`)**:
   - Pick the host operating system (`ubuntu-latest`, `windows-latest`, `macos-latest`).

3. **Configure the Job Matrix (`strategy.matrix`)**:
   - Test across multiple versions of languages or runtime environments simultaneously (e.g., Node 20.x, 22.x or Python 3.10, 3.11).

4. **Prepare Environment Steps**:
   - **Checkout Repository**: Pull source code into the runner (`actions/checkout@v4`).
   - **Setup Runtime**: Install and cache runtime environments (`actions/setup-node@v4`, `actions/setup-python@v5`, etc.).
   - **Install Dependencies**: Use deterministic install commands (e.g., `npm ci` instead of `npm install`).

5. **Execute Validation**:
   - Run linter checks, automated test suites, and build scripts.

6. **Deploy (CD Phase)**:
   - Use secure tokens stored in **GitHub Secrets** to authenticate with cloud services or registries.

---

# Part 2: Complete npm Publishing Pipeline (TypeScript/Node.js)

This section documents the exact automated workflow implemented for `mysql-craft`.

### 1. Project Configuration (`package.json`)

Ensure your `package.json` includes clean entry points and automated pre-publish hooks:

```json
{
  "name": "your-package-name",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist"
  ],
  "scripts": {
    "clean": "node -e \"require('fs').rmSync('dist', { recursive: true, force: true })\"",
    "build": "tsc -p tsconfig.build.json",
    "test": "vitest run",
    "prepublishOnly": "npm run clean && npm run build && npm run test"
  }
}
```

> **Note**: `package-lock.json` **must** be committed to Git. The `npm ci` command requires a lockfile to perform fast, reproducible dependency installations on CI runners.

---

### 2. Continuous Integration Workflow (`.github/workflows/ci.yml`)

This workflow automatically runs on every push or pull request to the `main` branch to catch broken tests or build issues early.

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    name: Build & Test (Node.js ${{ matrix.node-version }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [20.x, 22.x]

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Package
        run: npm run build

      - name: Run Test Suite
        run: npm run test
```

---

### 3. Continuous Deployment Workflow (`.github/workflows/publish.yml`)

This workflow triggers **only** when a new release is published on GitHub, compiling the project and deploying it directly to the npm registry.

```yaml
name: Publish to npm

on:
  release:
    types: [created]

jobs:
  publish:
    name: Publish Package
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22.x'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Package
        run: npm run build

      - name: Run Test Suite
        run: npm run test

      - name: Publish to npm
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

# Part 3: Managing Secrets & Security

Modern registries (like npm) mandate 2FA. To publish from automated environments, you must generate access tokens and store them securely in GitHub Secrets.

### Generating an npm Granular Access Token
1. Log in to [npmjs.com](https://www.npmjs.com/).
2. Navigate to **Account** -> **Access Tokens** -> **Generate New Token** -> **Granular Access Token**.
3. Set **Select packages** to **All packages** (or specific scoped packages).
4. Assign **Read and write** access for packages and scopes.
5. Click **Generate Token** and copy the string (`npm_xxxxxxxxxxxx`).

### Configuring GitHub Secrets
1. Open your GitHub Repository.
2. Go to **Settings** -> **Secrets and variables** -> **Actions**.
3. Click **New repository secret**.
4. Set **Name** to `NPM_TOKEN` (must match `${{ secrets.NPM_TOKEN }}` in your YAML).
5. Paste the token in **Secret** and save.

---

# Part 4: Release Workflow & Best Practices

Once your pipeline is set up, follow this standard deployment process for every future release:

### 1. Update Version
Bump the version number in `package.json` according to Semantic Versioning (SemVer):
- **Patch (`1.0.1`)**: Bug fixes.
- **Minor (`1.1.0`)**: New backward-compatible features.
- **Major (`2.0.0`)**: Breaking changes.

### 2. Commit and Push Changes
```bash
git add .
git commit -m "chore: bump version to 1.0.1"
git push origin main
```

### 3. Publish via GitHub Releases
1. Go to your GitHub repository -> **Releases** -> **Draft a new release**.
2. Create a tag matching your version (e.g., `v1.0.1`).
3. Set the release title and add changelog notes.
4. Click **Publish release**.

GitHub Actions will take over, execute tests, build your project, and publish the release live to npm!

---

### Verification
You can verify your live release via terminal:
```bash
npm view <package-name> version
```