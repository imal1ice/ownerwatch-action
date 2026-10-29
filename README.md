# ownerwatch admin-surface check

A GitHub Action that re-probes the admin surface of your EVM contracts and fails the job when anything changed since the report you committed: owner, proxy admin, implementation, AccessControl role holders, Safe signers and threshold, timelock delay. Built on [ownerwatch](https://github.com/imal1ice/ownerwatch).

Why: these change quietly. A signer rotates, a role gets granted for an integration and never revoked, a proxy admin moves. Nothing in your test suite notices. A weekly red check does.

## Usage

```yaml
# .github/workflows/admin-surface.yml
name: admin-surface
on:
  schedule:
    - cron: "17 6 * * 1"   # Mondays
  workflow_dispatch:
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: imal1ice/ownerwatch-action@v1
        with:
          config: ownerwatch.yaml        # the contracts and Safes to probe
          baseline: admin-surface.md     # the committed report to compare against
```

1. Write `ownerwatch.yaml` listing your contracts and Safes. `ownerwatch probe --rpc <rpc> --yaml <addresses>` generates one; see the [examples](https://github.com/imal1ice/ownerwatch/tree/main/examples).
2. Run the workflow once. It fails with "no baseline yet" and prints the report. Commit `admin-surface.new.md` as `admin-surface.md`.
3. From then on, the job is green while nothing changed and red with a diff in the job summary when something did. Commit the new report when the change is expected.

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `config` | `ownerwatch.yaml` | ownerwatch config to probe. Multi-chain configs produce one report section per chain. |
| `baseline` | `admin-surface.md` | Committed report to compare against. The fresh report is written next to it as `<baseline>.new.md`. |
| `label` | repository name | Heading of the report. |
| `rpc` | | RPC that allows `eth_call`, overriding the config (single-chain configs only). Multi-chain configs set `probe_rpc_url` per chain. |
| `roles-from-logs` | `false` | Replay role grants from logs to list every current role holder. Slower; works keyless on Ethereum, Optimism, Arbitrum, Gnosis and Polygon. |
| `fail-on-drift` | `true` | Fail the step on drift or missing baseline. Set `false` to only report. |
| `disclosure` | | Path to a committed [ERC-8241](https://ethereum-magicians.org/t/erc-8241-protocol-control-disclosure/28343) control disclosure (JSON). Checked against chain state; the step fails on any mismatch or undisclosed power. Needs `rpc`. |
| `disclosure-roles-from-logs` | `true` | Replay role grants so every role holder counts towards completeness. |
| `logs-rpc` | | RPC for the role replay's `eth_getLogs` (default: a public full-history gateway). |
| `version` | latest | Pin an ownerwatch version from PyPI. |

Drift is judged on the table of controllers, types and delays. The Findings section is shown in the summary but does not count, so rewording in a new ownerwatch release does not fail CI.

## ERC-8241 disclosures

If you publish an [ERC-8241](https://ethereum-magicians.org/t/erc-8241-protocol-control-disclosure/28343) control disclosure, commit it and let CI keep it honest:

```yaml
      - uses: imal1ice/ownerwatch-action@v1
        with:
          disclosure: erc-8241.json
          rpc: https://ethereum-rpc.publicnode.com
```

`ownerwatch disclosure draft <addresses> --rpc <rpc>` writes a starting disclosure from chain state. The check pins every read to one block and fails when a declared owner, admin, role, threshold, delay or edge no longer matches, or when the chain shows control over a declared component that the disclosure leaves out. Without `config`, only the disclosure is checked. See [docs/erc-8241.md](https://github.com/imal1ice/ownerwatch/blob/main/docs/erc-8241.md).

## Outputs

| Output | Meaning |
|---|---|
| `drifted` | `true` when the report differs from the baseline or no baseline exists. |
| `report` | Path of the freshly generated report, for `actions/upload-artifact` or a bot comment. |
| `disclosure-ok` | `true` when the ERC-8241 disclosure matches chain state with nothing undisclosed; empty without `disclosure`. |

## Notes

- Public RPCs are fine. The probe makes a few hundred `eth_call`s and no transactions; the examples use free endpoints.
- The comparison ignores the generated-on footer, so a report only differs when the admin surface does.
- For alerts within a block instead of a weekly check, run [ownerwatch](https://github.com/imal1ice/ownerwatch) itself.

MIT. Contact: imal1ice@proton.me.
