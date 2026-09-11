# Carabiner Labs Actions

Actions and reusable workflows currently in development.

## OSPS Compliance

`.github/workflows/osps-compliance.yaml` is a reusable workflow that gathers
signed evidence about a repository and evaluates it against an
[OSPS Baseline](https://baseline.openssf.org/) policy with 🔴🟡🟢 AMPEL. Every
push produces one `attestations.jsonl` with the evidence, a signed policy
evaluation result and an evaluation report in the job summary.

The evidence it collects, all signed with the workflow's own identity:

| Evidence | Produced by |
| --- | --- |
| SPDX SBOM of the source | `sbom/source` |
| `SECURITY-INSIGHTS.yml` and `.openeox.json`, attested to the commit | `bnd/commit` |
| Organization, repository and branch rule settings | `snappy/snap` |
| OpenSSF Scorecard results | `scorecard` + `bnd/sign` |
| Test results (Go modules) | `beaker/tests` |
| OSV scan results | `osv-scanner` + `bnd/sign` |
| OpenVEX document of the branch | `vexflow/assemble` |
| SLSA source provenance | fetched from the repository's `SLSA Source` workflow |

### Usage

Call the workflow from a workflow of your own triggered on push. The job
needs `id-token: write` to sign, `contents: read` to check out the code and
`actions: read` to fetch the source provenance from another run:

```yaml
name: OSPS Compliance

on:
  push:

permissions: {}

jobs:
  compliance:
    permissions:
      id-token: write
      contents: read
      actions: read
    uses: carabiner-labs/actions/.github/workflows/osps-compliance.yaml@main
```

Pin the workflow to a commit instead of `main` once you want it to stop
moving under you.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `policy` | the OSPS Baseline policy set | AMPEL policy or policy set to evaluate the evidence against. |
| `branch` | the repository's default branch | Branch whose protection rules and VEX document are attested. |
| `slsa-source-workflow` | `SLSA Source` | Name of the workflow that computes the SLSA source provenance for each push. Set to an empty string to skip fetching it. |
| `fail-on-policy` | `false` | Fail the job when the policy evaluation does not pass. |
| `push-attestations` | `false` | Push the evidence and the evaluation results to the repository's GitHub attestations store. The calling job must also grant `attestations: write`. |

```yaml
    uses: carabiner-labs/actions/.github/workflows/osps-compliance.yaml@main
    with:
      branch: main
      fail-on-policy: true
```

### What the repository needs

- **A `.vexflow` repository in the organization**, where vexflow keeps the
  vulnerability triage. The VEX document is assembled from it.
- **Dependency manifests** that OSV can scan (`go.mod`, `package.json`, and
  so on). Go modules also get their tests attested with beaker.
- **A `SLSA Source` workflow** producing the `prov_metadata` artifact, as
  set up by
  [slsa-framework/source-actions](https://github.com/slsa-framework/source-actions).
  Without one, the evaluation runs without source provenance; set
  `slsa-source-workflow: ''` to skip the lookup altogether.
- Optionally, a `SECURITY-INSIGHTS.yml` and a `.openeox.json`. They are
  attested when present and skipped otherwise.

### Results

- The `attestations.jsonl` artifact holds all the evidence, one attestation
  per line, ready for `ampel verify --collector jsonl:attestations.jsonl`.
- The `ampel.intoto.json` artifact is the signed evaluation result.
- With `push-attestations: true`, a second job pushes every attestation in
  both artifacts to the repository's GitHub attestations store with
  `bnd push`. It fails with an explanation when the calling job did not grant
  `attestations: write`.
- The job summary shows the evaluation report. The job itself only fails on
  a policy failure when `fail-on-policy` is set.

The `carabiner-dev/actions` references in the workflow are pinned to a commit
on `main` until the actions it uses are part of a release.
