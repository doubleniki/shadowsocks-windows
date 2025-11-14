# GitHub Actions Workflows Documentation

This document describes the CI/CD workflows configured for the Shadowsocks-Windows project.

## Overview

The project uses GitHub Actions for continuous integration, testing, security analysis, and automated releases. This replaces and extends the previous AppVeyor-based CI system.

## Workflows

### 1. Build and Test (`build.yml`)

**Triggers:**
- Push to `master` or `develop` branches
- Pull requests to `master` or `develop`
- Manual dispatch

**What it does:**
- Builds the solution for both x86 and x64 platforms
- Runs automated tests using MSTest
- Generates file hashes (MD5, SHA-1, SHA-256, SHA-512)
- Creates ZIP archives with release artifacts
- Uploads test results and build artifacts

**Platforms:** x86, x64
**Configurations:** Debug, Release

**Artifacts:**
- `Shadowsocks-x86.zip` - 32-bit build
- `Shadowsocks-x64.zip` - 64-bit build
- Hash files for verification
- Test results (TRX format)

### 2. CodeQL Security Analysis (`codeql.yml`)

**Triggers:**
- Push to `master` or `develop`
- Pull requests to `master`
- Weekly schedule (Mondays at 00:00 UTC)
- Manual dispatch

**What it does:**
- Performs static code analysis for security vulnerabilities
- Checks for common security issues and code quality problems
- Runs extended security queries
- Reports findings in GitHub Security tab

**Analysis type:** security-extended, security-and-quality

### 3. Release (`release.yml`)

**Triggers:**
- Push of version tags (v*)
- Manual dispatch

**What it does:**
- Builds release versions for x86 and x64
- Calculates file hashes
- Generates changelog from git commits
- Creates ZIP packages with localization files
- Automatically creates GitHub Release with:
  - Generated changelog
  - Release notes
  - Downloadable artifacts
  - Hash files for verification

**Artifacts created:**
- `Shadowsocks-{version}-x86.zip`
- `Shadowsocks-{version}-x86.zip.hash`
- `Shadowsocks-{version}-x64.zip`
- `Shadowsocks-{version}-x64.zip.hash`

**Release types:**
- Standard releases for normal tags
- Pre-releases for tags containing: alpha, beta, rc

### 4. Pull Request Checks (`pr-checks.yml`)

**Triggers:**
- Pull request opened, synchronized, or reopened

**What it does:**
- Quick build validation (Debug and Release x86)
- Code validation checks:
  - Scans for new TODOs in critical files
  - Checks for common security anti-patterns
  - Detects potential hardcoded credentials
  - Identifies broad exception catching
- Runs unit tests
- Posts summary comment on PR

**Benefits:**
- Fast feedback for contributors
- Prevents common issues before merge
- Automated code review assistance

## Dependabot Configuration

File: `.github/dependabot.yml`

**What it does:**
- Automatically checks for NuGet package updates weekly
- Monitors GitHub Actions versions
- Creates pull requests for dependency updates
- Groups minor and patch updates together

**Schedule:** Weekly on Mondays at 09:00

**Labels applied:**
- NuGet updates: `dependencies`, `nuget`
- Actions updates: `dependencies`, `github-actions`

## Migration from AppVeyor

### What's New

1. **Extended platform support:** Now builds both x86 and x64 (AppVeyor only did x86)
2. **Automated testing:** Tests are now run automatically on every build
3. **Security scanning:** CodeQL analyzes code for vulnerabilities
4. **Dependency management:** Dependabot automatically updates packages
5. **PR automation:** Automated checks and comments on pull requests
6. **Better caching:** Improved NuGet package caching

### What's Improved

1. **Build speed:** Parallel matrix builds for different configurations
2. **Artifact retention:** Configurable retention periods (90 days for releases)
3. **Release automation:** Automatic changelog generation and GitHub Releases
4. **Test reporting:** Structured test results with artifacts

### Backwards Compatibility

- AppVeyor configuration is still present and can run in parallel
- Same hash algorithms (MD5, SHA-1, SHA-256, SHA-512)
- Similar ZIP packaging structure
- Compatible artifact naming

## Usage

### For Contributors

**When you create a PR:**
1. The `pr-checks` workflow will automatically run
2. Wait for all checks to pass (green checkmarks)
3. Review any automated comments on your PR
4. Fix any issues found by the validation checks

**What to expect:**
- Build validation (5-10 minutes)
- Code validation results
- Test execution summary
- Automated PR comment with results

### For Maintainers

**Creating a Release:**

1. Create and push a version tag:
   ```bash
   git tag v4.4.2
   git push origin v4.4.2
   ```

2. The release workflow will automatically:
   - Build both x86 and x64 versions
   - Generate changelog
   - Create GitHub Release
   - Upload artifacts

3. Review and publish the draft release if needed

**Monitoring Security:**

- Check the Security tab for CodeQL findings
- Review Dependabot PRs for dependency updates
- Monitor weekly CodeQL scans

**Managing Dependencies:**

- Dependabot will create PRs for updates
- Review and merge security updates promptly
- Test dependency updates before merging

## Troubleshooting

### Build Failures

1. Check the Actions tab for detailed logs
2. Look for NuGet restore issues
3. Verify MSBuild errors in the log
4. Check platform-specific issues (x86 vs x64)

### Test Failures

1. Review test results artifacts
2. Check TRX files for detailed test output
3. Run tests locally to reproduce
4. Verify test assembly path is correct

### CodeQL Issues

1. Review findings in Security > Code scanning
2. Check severity and confidence levels
3. Fix or dismiss false positives
4. Re-run analysis after fixes

## Best Practices

1. **Always run tests locally** before pushing
2. **Keep dependencies updated** - review Dependabot PRs regularly
3. **Monitor security alerts** - check CodeQL results
4. **Use semantic versioning** for tags (v1.2.3)
5. **Write meaningful commit messages** - they appear in changelogs
6. **Fix TODO comments** in critical files before merging

## Maintenance

### Updating Workflows

1. Edit workflow files in `.github/workflows/`
2. Test changes on a feature branch first
3. Use `workflow_dispatch` for manual testing
4. Monitor first run carefully
5. Update this documentation

### Updating Dependencies

**NuGet packages:**
- Dependabot handles this automatically
- Can also manually update `packages.config`

**GitHub Actions:**
- Dependabot monitors action versions
- Update to latest stable versions
- Test compatibility after updates

## Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [CodeQL Documentation](https://codeql.github.com/docs/)
- [Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
- [MSBuild Reference](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild)

## Support

For issues with CI/CD:
1. Check workflow run logs in Actions tab
2. Review this documentation
3. Open an issue with the `ci/cd` label
4. Provide workflow run URL and error logs
