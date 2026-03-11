# cc-edge-gemini-antigravity-io

Cribl Edge pack for Gemini CLI and Antigravity IDE telemetry collection.

## Version Policy

This pack uses **semantic versioning** starting from `0.x.x` (pre-release). It has not reached a stable `1.0.0` release yet.

### Rules for AI agents

- **Patch bumps** (`0.x.Y → 0.x.Y+1`): AI may bump for bug fixes, performance improvements, and minor corrections.
- **Minor bumps** (`0.X.y → 0.X+1.0`): AI may bump for new features, new inputs, or non-breaking changes.
- **Major bumps** (`0.x.y → 1.0.0` or `N.x.y → N+1.0.0`): **Human only.** Never bump the major version without explicit human approval. The jump from `0.x.x` to `1.0.0` signals production readiness and must be a deliberate decision.

### Release workflow

1. Make changes on a feature branch
2. Update `package.json` version (minor or patch only)
3. Update the `## Release Notes` section in `README.md` with the new version entry
4. Merge PR to main
5. Create GitHub release with the `.crbl` artifact built locally:
   ```sh
   tar -czf cc-edge-gemini-antigravity-io-vX.Y.Z.crbl data default package.json README.md
   cp cc-edge-gemini-antigravity-io-vX.Y.Z.crbl cc-edge-gemini-antigravity-io.crbl
   gh release create vX.Y.Z --draft <files> && gh release edit vX.Y.Z --draft=false
   ```
   Note: Create as draft with assets first, then publish. Publishing a release before uploading assets makes it immutable.

## File Operations

- Read files with the Read tool (NEVER cat, head, tail)
- Edit existing files with the Edit tool (NEVER sed, awk)
- Create new files with the Write tool (NEVER echo >, heredocs)
- Search file contents with the Grep tool
- Find files with the Glob tool
