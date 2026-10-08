# Dependency Verification Guide

## Overview
Comprehensive guide for verifying dependencies in the langflow project. This document provides processes, tools, and best practices for ensuring dependency integrity and security.

## Dependency Management Strategy

### Core Principles
1. **Pinned Versions** - Lock exact versions in production
2. **Security First** - Regular vulnerability scanning
3. **Minimal Dependencies** - Only include necessary packages
4. **Regular Updates** - Keep dependencies current

## Verification Checklist

### Pre-Deployment Verification
```bash
#!/bin/bash
set -e

echo "🔍 Verifying dependencies..."

# 1. Check for vulnerabilities
npm audit || exit 1

# 2. Verify all dependencies are installed
npm ci --audit

# 3. Check for outdated packages
npm outdated

# 4. Verify licenses
npm ls --depth=0

# 5. Test dependency compatibility
npm run test:deps

echo "✅ All dependency checks passed"
```

## Python Dependencies

### Requirements Analysis

```python
# requirements.txt verification script
import subprocess
import sys
from pathlib import Path

def verify_requirements():
    """Verify all Python requirements are properly specified"""
    
    # 1. Check syntax
    result = subprocess.run(
        ["pip", "install", "--dry-run", "-r", "requirements.txt"],
        capture_output=True
    )
    if result.returncode != 0:
        print("❌ Invalid requirements.txt syntax")
        return False
    
    # 2. Verify versions
    with open("requirements.txt") as f:
        for line in f:
            if line.strip() and not line.startswith("#"):
                package = line.split("==")[0].split(">")[0].split("<")[0]
                try:
                    subprocess.run(
                        ["pip", "show", package],
                        capture_output=True,
                        check=True
                    )
                except subprocess.CalledProcessError:
                    print(f"❌ Package not found: {package}")
                    return False
    
    print("✅ All requirements verified")
    return True

if __name__ == "__main__":
    sys.exit(0 if verify_requirements() else 1)
```

### Common Python Issues

```python
# ✅ Correct pinned versions
# requirements.txt
langflow==1.0.0
pydantic==2.1.0
fastapi==0.95.0
pyyaml==6.0
requests==2.31.0

# ❌ Avoid floating versions
# langflow>=1.0.0  (too loose)
# langflow~=1.0    (unpredictable)
```

## Node.js Dependencies

### Package Lock Verification

```json
{
  "name": "langflow-ui",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "packages": {
    "": {
      "name": "langflow-ui",
      "version": "1.0.0",
      "dependencies": {
        "react": "18.2.0",
        "react-dom": "18.2.0"
      }
    }
  }
}
```

### Verification Script

```bash
#!/bin/bash

echo "Verifying npm dependencies..."

# 1. Check lock file integrity
npm ci --audit 2>&1 || {
    echo "❌ Lock file integrity check failed"
    exit 1
}

# 2. Audit for vulnerabilities
npm audit 2>&1 | grep -E "(high|critical)" && {
    echo "❌ Security vulnerabilities found"
    exit 1
} || echo "✅ No critical vulnerabilities"

# 3. Check for duplicate packages
npm dedupe --dry-run 2>&1 | grep "added" && {
    echo "⚠️ Duplicate packages detected"
    npm dedupe
} || echo "✅ No duplicates"

echo "✅ All npm checks passed"
```

## Container Dependency Verification

### Docker Best Practices

```dockerfile
# ✅ Good: Specific base image version
FROM python:3.11.4-slim

# ✅ Pin all package versions
RUN pip install --no-cache-dir \
    langflow==1.0.0 \
    pydantic==2.1.0 \
    fastapi==0.95.0

# ✅ Verify installation
RUN python -c "import langflow; print(langflow.__version__)"

# ✅ Non-root user for security
USER langflow
```

### Container Scanning

```bash
#!/bin/bash
# Scan built image for vulnerabilities

docker build -t langflow:latest .

# Using Trivy (recommended)
trivy image langflow:latest

# Using Snyk
snyk container test langflow:latest
```

## Security Scanning

### Automated Vulnerability Detection

```bash
#!/bin/bash
echo "🔒 Running security scans..."

# 1. Python: bandit
bandit -r src/ || true

# 2. Node.js: npm audit
npm audit --audit-level=high || true

# 3. Dependencies: safety
pip install safety
safety check

# 4. SBOM generation
cyclonedx-bom -o sbom.xml

echo "✅ Security scanning complete"
```

### Dependency License Compliance

```bash
#!/bin/bash
# Check license compliance

# Python
pip install pip-licenses
pip-licenses --format=csv > licenses.csv

# Node.js
npm install -g license-report
license-report --package=./package.json

# Check for problematic licenses
grep -i "GPL\|AGPL" licenses.csv && {
    echo "⚠️ GPL-like licenses detected"
} || echo "✅ All licenses compliant"
```

## Update Strategy

### Dependency Update Process

```bash
#!/bin/bash
set -e

branch="deps/$(date +%Y-%m-%d)"
git checkout -b $branch

# 1. Update Python dependencies
pip install --upgrade pip
pip install --upgrade -r requirements.txt
pip freeze > requirements.txt

# 2. Update Node.js dependencies
npm update
npm ci

# 3. Run tests
npm run test
pip -m pytest

# 4. Create PR
git add .
git commit -m "chore: update dependencies"
git push origin $branch
```

### Major Version Updates

```bash
#!/bin/bash
# Handle breaking changes carefully

# 1. Create feature branch
git checkout -b deps/major-update

# 2. Update package
npm install package@latest
# or
pip install --upgrade package

# 3. Run comprehensive tests
npm run test:full
python -m pytest -v

# 4. Update documentation
# - API changes
# - Migration guide
# - Deprecations

# 5. Create PR with detailed notes
```

## Monitoring & Alerts

### Continuous Monitoring

```yaml
# .github/workflows/dependency-check.yml
name: Dependency Verification

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  push:
    paths:
      - 'requirements.txt'
      - 'package.json'
      - 'package-lock.json'

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check Python dependencies
        run: |
          pip install -r requirements.txt
          pip install safety
          safety check
      
      - name: Check Node dependencies
        run: |
          npm ci
          npm audit --audit-level=high
      
      - name: License compliance
        run: |
          npm install -g license-report
          license-report --package=./package.json
```

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Dependency conflicts | Version mismatch | Use exact pinning or constraint solver |
| Missing dependencies | Incomplete lock file | Regenerate lock file and commit |
| Security vulnerabilities | Old versions | Update to patched versions |
| Build failures | Incompatible versions | Run compatibility tests locally |
| Performance degradation | Heavy dependencies | Profile and optimize imports |

## Tools Reference

### Python
- **pip-audit**: Vulnerability scanning
- **safety**: Dependency security
- **pipdeptree**: Dependency visualization
- **pip-licenses**: License verification

### Node.js
- **npm audit**: Built-in vulnerability scanner
- **snyk**: Advanced security scanning
- **npm-check-updates**: Update checker
- **license-report**: License compliance

### Container
- **Trivy**: Container image scanning
- **Snyk Container**: Container security
- **Grype**: SBOM generation and scanning

## Best Practices

1. **Automate Verification** - Run checks in CI/CD
2. **Keep Updated** - Regular dependency updates
3. **Monitor Security** - Subscribe to advisories
4. **Test Thoroughly** - Verify after updates
5. **Document Changes** - Track dependency updates
6. **Use Lock Files** - Ensure reproducible builds
7. **Minimize Dependencies** - Only use what's needed

## Resources

- [npm Security](https://docs.npmjs.com/packages-and-modules/npm-audit)
- [Python Package Index](https://pypi.org/)
- [Security.md Standard](https://securitymd.org/)
- [OWASP Dependency Check](https://owasp.org/www-project-dependency-check/)
- [CycloneDX](https://cyclonedx.org/)
