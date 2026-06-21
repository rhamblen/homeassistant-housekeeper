# Release process

Versioning: [Semantic Versioning](https://semver.org). Changelog:
[Keep a Changelog](https://keepachangelog.com). Phases map loosely to minor versions
(Phase 1 → v0.1.0, etc.); the repo scaffold is v0.1.0-pre.

When asked to publish a version:

1. **Commit** staged work: `git commit -m "feat(vX.Y.Z): ..."`.
2. **Update notes first** — `CHANGELOG.md` must match the tag *before* tagging.
3. **Tag**: `git tag -a vX.Y.Z -m "vX.Y.Z — short description"`.
4. **Push** branch + tag: `git push origin <branch> --tags`.
5. **Create the GitHub Release explicitly** (tag+push alone shows nothing on the
   Releases page):
   - With `gh`: `gh release create vX.Y.Z --title "..." --notes-file RELEASE_NOTES.md`
   - Without `gh` (use the token in Git Credential Manager — never print it): see the
     PowerShell REST snippet in the user's global guidance.

Releases are published **only when explicitly asked**.

> First publish still needs: chosen repo name + visibility, GitHub remote created, and
> explicit go-ahead to push (pushing is an outward-facing action).
