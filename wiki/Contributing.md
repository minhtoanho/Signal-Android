# Contributing

Thank you for your interest in contributing to Signal Android!

## Before You Start

### Code of Conduct

Be respectful and constructive in all interactions. Signal is used by millions of people worldwide who rely on it for secure communication.

### Development Ideology

1. **The answer is not more options.** If you feel compelled to add a preference that's exposed to the user, it's very possible you've made a wrong turn somewhere.

2. **The user doesn't know what a key is.** We need to minimize the points at which a user is exposed to cryptographic terminology.

3. **There are no power users.** The idea that some users "understand" concepts better than others has proven to be, for the most part, false.

4. **If it's "like PGP," it's wrong.** PGP is our guide for what not to do.

5. **It's an asynchronous world.** Be wary of anything that is anti-asynchronous: ACKs, protocol confirmations, or any protocol-level "advisory" message.

6. **There is no such thing as time.** Protocol ideas that require synchronized clocks are doomed to failure.

## Getting Started

### 1. Fork and Clone

```bash
git clone https://github.com/YOUR_USERNAME/Signal-Android.git
cd Signal-Android
```

### 2. Set Up Development Environment

See [Building.md](Building.md) for detailed setup instructions.

### 3. Build and Run

```bash
./gradlew assemblePlayProdDebug
```

## Finding Issues to Work On

### Good First Issues

Look for issues labeled:
- `good first issue`
- `help wanted`
- `bug` (simple ones)

### Issue Tracker

https://github.com/signalapp/Signal-Android/issues

### Before Starting

1. Check if issue is already assigned
2. Comment on the issue to discuss your approach
3. Wait for feedback before starting work

## Submitting Changes

### Contributor License Agreement

You must [sign the CLA](https://signal.org/cla/) before your PR can be merged.

### Pull Request Process

1. **Create a Branch**

```bash
git checkout -b feature/your-feature-name
```

2. **Make Changes**

- Follow [Code Style Guidelines](Code-Style-Guidelines.md)
- Write tests for new functionality
- Update documentation if needed

3. **Run Quality Checks**

```bash
# Format code
./gradlew format

# Run full QA suite
./gradlew qa
```

4. **Commit Changes**

Write clear, descriptive commit messages:

```
Fix message delivery race condition

- Add synchronization to message queue
- Update tests for concurrent delivery
- Fixes #1234
```

5. **Push and Create PR**

```bash
git push origin feature/your-feature-name
```

Create a pull request with:
- Description of changes
- Link to related issue
- Testing instructions

### PR Guidelines

#### Smaller is Better

Big changes are significantly less likely to be accepted. Large features often require:
- Protocol modifications
- Multi-platform coordination
- Staged rollout process

Start with small, simple PRs to become familiar with the codebase.

#### Complete and Tested

- PRs should be ready to merge
- Include tests
- Pass all CI checks
- No work-in-progress PRs

#### One Thing Per PR

- One feature/fix per PR
- Easier to review
- Faster to merge

## Code Review

### What to Expect

- Reviews may take time due to team size
- You may be asked for changes
- Not all PRs will be merged

### After Review

1. Address feedback promptly
2. Push new commits (don't force push during review)
3. Mark conversations as resolved
4. Re-request review when ready

## Reporting Issues

### Before Reporting

1. Search existing issues (open and closed)
2. Try the latest version
3. Check [Signal Support](https://support.signal.org/)

### Bug Reports Should Include

1. **Description**: What happened vs. what you expected
2. **Steps to Reproduce**: Clear, numbered steps
3. **Environment**: Android version, device, Signal version
4. **Logs**: Debug logs (Settings → Advanced → Submit Debug Log)
5. **Screenshots**: If applicable

### What NOT to Report

- Feature requests → Use [community forum](https://community.signalusers.org/c/feature-requests)
- Support questions → Contact [support@signal.org](mailto:support@signal.org)
- Security issues → Email [security@signal.org](mailto:security@signal.org)

## Translations

Help translate Signal:

https://community.signalusers.org/c/translation-feedback/

## Ways to Contribute

- **Code**: Fix bugs, add features
- **Documentation**: Improve wiki, code comments
- **Translations**: Help localize Signal
- **Testing**: Test PRs, report bugs
- **Support**: Help other users in the forum
- **Donations**: [Support Signal](https://signal.org/donate/)

## Getting Help

- **Community Forum**: https://community.signalusers.org
- **Signal Support**: https://support.signal.org
- **Email**: support@signal.org

## License

By contributing, you agree that your contributions will be licensed under the GNU AGPLv3 license.