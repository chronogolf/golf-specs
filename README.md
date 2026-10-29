# Golf OpenSpec Store

This repository is the shared [OpenSpec](https://openspec.dev/) store for the
Golf spec-driven development (SDD) pilot. It holds the specifications and
change records for pilot work; it does **not** contain the source code being
changed.

The pilot uses one shared store so a change can have a single plan and set of
behavioral specifications even when its implementation spans several Golf
repositories. Source code, tests, and implementation pull requests remain in
their respective repositories.

## Why this repository exists

Golf is evaluating SDD as a way to make intent, design decisions, and expected
behavior explicit before implementation begins. OpenSpec was selected for the
pilot because it supports a lightweight workflow and can keep shared planning
artifacts outside of application repositories.

Using this repository as an OpenSpec store lets the pilot:

- maintain one authoritative change record for work across multiple
  repositories;
- preserve the specifications produced by completed changes as reusable
  capability documentation;
- let developers adopt the workflow without adding OpenSpec planning artifacts
  or CI changes to every Golf repository; and
- evaluate the value and friction of SDD before making longer-term process or
  tooling decisions.

This is an experimental pilot. OpenSpec stores are a beta feature, and the
workflow may change as the pilot and OpenSpec evolve.

## Repository layout

```text
.
├── .openspec-store/
│   └── store.yaml          # Identifies this store as `golf-specs`
└── openspec/
    ├── config.yaml         # Store-level OpenSpec configuration
    ├── changes/            # Active change proposals and their artifacts
    │   └── archive/        # Completed changes
    └── specs/              # Long-lived capability specifications
```

An active change normally captures its proposal, design, tasks, and one or
more behavior specifications. When the change is archived, its capability
specifications are collected under `openspec/specs/` and its change record is
preserved under `openspec/changes/archive/`.

## Get started

### Prerequisites

Install the OpenSpec CLI and the integration for the AI development tool you
use. See the [OpenSpec installation documentation](https://openspec.dev/docs/installation)
for the current instructions.

### Clone and register the store

Each participant registers their local clone once. Replace the placeholder
with this repository's clone URL and choose a suitable local location.

```bash
git clone <golf-specs-repository-url> ~/openspec/golf-specs
openspec store register ~/openspec/golf-specs
```

Confirm that OpenSpec can find the shared store from any directory:

```bash
openspec status --store golf-specs
```

The output should identify `golf-specs` as the OpenSpec root. If it does not,
run `openspec doctor` for setup diagnostics.

### Connect a participating code repository

For a **store-only** setup, add the following to the participating code
repository's `openspec/config.yaml` and commit it with that repository:

```yaml
store: golf-specs
```

With this configuration, OpenSpec commands run from the code repository use
this store automatically. The local `openspec/` directory should contain only
`config.yaml`; local `specs/` or `changes/` directories take precedence over
the `store:` setting.

Install the AI-tool integration from the root of the code repository as
needed. This creates the tool-specific integration files in the code
repository while keeping change artifacts in this store. Refer to the
[OpenSpec store setup guidance](https://openspec.dev/docs/stores#install-integrations-in-a-store-only-repo)
for the current command for your tool.

If the code repository is deliberately using its own OpenSpec artifacts as
well, use this store explicitly with `--store golf-specs` for shared work
instead of treating it as store-only.

## Working on a pilot change

The pilot's default is a one-to-one relationship between a Jira Story, Task,
or Bug and an OpenSpec change. Use a descriptive, kebab-case change ID, record
the Jira ticket in the change artifacts, and link all implementation pull
requests and verification evidence before archiving.

1. **Start in sync.** Pull the latest version of this repository before
   creating or updating a change.
2. **Explore.** Use `/opsx-explore` to understand the problem, affected code
   repositories, dependencies, and existing behavior.
3. **Propose.** Use `/opsx-propose` to create the named change and draft its
   `proposal.md`, `design.md`, `tasks.md`, and behavior specifications. Review
   and refine these artifacts before implementation.
4. **Implement.** Use `/opsx-apply` to complete the agreed tasks in the
   applicable source repositories. Keep implementation pull requests linked
   from the change record.
5. **Verify.** Use `/opsx-verify` to confirm that the implementation matches
   the specification and record the evidence.
6. **Archive.** Use `/opsx-archive` after implementation and review are
   complete. This preserves the completed change and promotes its capability
   specifications for future reference.

When a repository has been configured as store-only, the workflow commands
will route artifacts here. To work with the store directly, or to make the
target unambiguous, use the store name:

```bash
openspec status --store golf-specs
openspec new change <change-id> --store golf-specs
openspec status --change <change-id> --store golf-specs
```

OpenSpec prints the selected root before command output. Check that it says
`golf-specs` before creating or editing shared artifacts.

## Cross-repository changes

A shared store is most valuable when a feature affects more than one code
repository. Create **one** change here rather than separate, disconnected
plans in each implementation repository. The change should identify:

- every affected repository and the responsibility it has in the change;
- cross-repository interfaces, sequencing, and dependencies;
- implementation pull requests for each repository; and
- end-to-end verification evidence for the intended behavior.

The shared specification describes the behavior and coordination. It does not
replace repository-specific code review, testing, or release practices.

## Sharing and reviewing artifacts

This store is a regular Git repository. Treat its changes like code changes:

- work on a branch when collaborating on a change;
- commit and push artifacts so other participants can see them;
- review store pull requests alongside the relevant implementation pull
  requests; and
- pull the store regularly, especially before starting work or archiving a
  change.

OpenSpec does not clone, pull, push, or merge the store for you. A local clone
only reflects the specs and changes that have been fetched from Git.

## Pilot feedback

For each completed change, the developer and reviewer should record concise
feedback on:

- **Clarity:** whether the specification exposed an ambiguity or dependency
  before coding;
- **Coordination:** whether it identified affected repositories and surfaced
  dependencies early enough;
- **Verification:** whether the change links its Jira ticket, implementation
  pull requests, and evidence of the intended behavior; and
- **Value versus effort:** whether the workflow was worth repeating for a
  similar change and what should be simplified.

These observations support the pilot's evaluation of whether SDD helps Golf
coordinate and deliver changes effectively.

## Further reading

- [Spec-Driven Development Pilot initiative](https://lightspeedhq.atlassian.net/wiki/spaces/CG/pages/3461546051/Spec-Driven+Development+Pilot)
- [Golf Spec-driven Development pilot RFC](../documents/rfc/spec-driven-dev-pilot.md)
  (available in the local development workspace)
- [OpenSpec stores documentation](https://openspec.dev/docs/stores)
- [OpenSpec workflow documentation](https://openspec.dev/docs)
