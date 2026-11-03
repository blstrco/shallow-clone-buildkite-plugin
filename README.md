## Usage

### Basic shallow clone (original behavior)

```yml
steps:
  - command: buildkite-agent pipeline upload
    plugins:
      - blstrco/shallow-clone#v0.0.1:
          depth: 1
```

### Bypassing git mirrors for upload steps (v0.1.0+)

For Buildkite agent v4.x with git mirrors enabled, the standard shallow-clone flags (`BUILDKITE_GIT_CLONE_FLAGS`, `BUILDKITE_GIT_FETCH_FLAGS`) **cannot** override mirror behavior. The agent's mirror configuration is authoritative, so cold checkouts still create full bare mirrors (which can be multi-GB for large repos).

The `bypass-mirrors` option solves this by:
1. Skipping the agent's default mirrored checkout entirely (`BUILDKITE_SKIP_CHECKOUT=true`)
2. Performing a **blobless** clone (`--filter=blob:none`) that downloads trees and commits but not blob content
3. Fetching only the necessary refs for the build commit and default branch
4. Preserving `if_changed` functionality by ensuring `origin/<default-branch>` is available for merge-base calculations

This is ideal for **pipeline upload steps** where you only need repository structure and history for path-based conditionals, not file contents.

#### Example: Upload step avoiding cold mirror cost

```yml
steps:
  - label: ":pipeline: Upload dynamic pipeline"
    command: buildkite-agent pipeline upload .buildkite/pipelines/events_ingestion.yml
    plugins:
      - blstrco/shallow-clone#v0.1.0:
          bypass-mirrors: true
          depth: 50  # optional: limit history depth (compatible with blobless)
```

#### Expected behavior

- **Without `bypass-mirrors`**: Cold agent creates full mirror in `/var/lib/buildkite-agent/git-mirrors/` (e.g., 2.31 GiB / ~94s for dp-ingestion)
- **With `bypass-mirrors: true`**: Blobless clone to build directory only (~tens of MB, ~10-15s)
- `if_changed` pipeline conditionals continue to work via `BUILDKITE_FETCH_DIFF_BASE=true` and fetched default branch ref

#### Caveats

- This option is designed for **upload steps** and other jobs that don't need file contents
- Regular command steps that build/test code should use standard checkout (mirrors are beneficial for warm agents)
- The `bypass-mirrors` path always does a fresh clone (no mirror reuse on subsequent builds)
