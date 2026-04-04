# secrets-gitleaks

Composite GitHub Action that runs [Gitleaks](https://github.com/gitleaks/gitleaks) with a **safe** CI contract. Each match is a **secret** finding; the JSON-driven output uses **`secret_count`**.

- **0 findings** → success (and `scan_status=clean`).
- **≥ 1 finding** → failure when `fail_on_findings` is `true` (`scan_status=findings_found`).
- Logs include **count**, **rule id**, **file**, and **line** only — **not** secret values (`--redact` plus structured summary without `Secret` fields).

## Caller responsibilities

- Run on **`ubuntu-latest`** (Linux x64 binary). **`jq`** must be available (included on GitHub-hosted runners; install it on self-hosted runners).
- The action downloads the official Gitleaks release tarball with **`curl -fSL`** (HTTP failures do not leave a bogus binary). For maximum supply-chain assurance, pin **`gitleaks_version`** and verify the release **checksum** from the upstream project if your policy requires it (not automated in this action today).
- **`actions/checkout`** must run **before** this action. For **full git history** scans, use `fetch-depth: 0` on checkout; shallow clones limit what Gitleaks can see in history.
- With default **`no_git: false`**, `working_directory` must be a **git** checkout. With **`no_git: true`**, a normal directory tree is enough.

## Exit behavior

Gitleaks is invoked with **`--exit-code 0`** so **findings do not cause a non-zero process exit** from Gitleaks itself. **Any non-zero exit code** from the scanner process therefore indicates a **scanner or environment error**, not “how many secrets were found.” The action sets **`scan_status`** and **`secret_count`** from the JSON report and applies **`fail_on_findings`** itself (that step may still exit **1** when findings exist and failure is enabled). Log output lists at most **20** findings (rule, file, line); it does not print secret values.

## Removed the file, but CI still fails?

With **`no_git: false`** (the default), Gitleaks scans **git history** (every scanned commit), not only the files at **HEAD**. If a secret was ever committed, a later commit that **deletes** that file does **not** erase the earlier commit; the scanner will still report the finding. That is intentional: the value remains reachable via older SHAs, forks, and clones.

**To get CI green again** (after handling any **real** credential with **revocation/rotation**):

- **Rewrite history** so no commit contains the secret (for example **squash** the bad commits into one clean commit, or use **`git filter-repo`** / BFG to strip the path), then **force-push** with care and team coordination.
- Or maintain a **`baseline_path`** report that lists known historical findings you are cleaning up gradually (see Gitleaks docs).

**To avoid this when testing:** use a **throwaway branch** and delete it after the demo, or never push the test secret to a shared branch.

## False positives and allowlisting

**Do not use `.gitignore`** to try to fix Gitleaks results: ignored files may still be in **git history**, and hiding paths is not the right control for “this match is OK.”

Use instead:

- **`.gitleaksignore`** — fingerprints from a report (see Gitleaks docs).
- **`.gitleaks.toml`** — allowlists / custom rules; pass via `config_path` if not at repo root.
- Inline **`# gitleaks:allow`** on a line when you intentionally commit a test secret (document why).

**Baselines vs ignore files:** For large repositories with **lots of historical noise**, a **`baseline_path`** report is often the right way to phase remediation. **`.gitleaksignore`** (fingerprints) fits **targeted** suppressions for specific lines or files you have triaged.

## Inputs

| Input | Default | Description |
|--------|---------|-------------|
| `gitleaks_version` | `8.18.4` | Release version (no `v` prefix). |
| `working_directory` | `.` | Scan root (relative to workspace). |
| `config_path` | *(empty)* | Optional config relative to `working_directory`. |
| `baseline_path` | *(empty)* | Optional baseline report relative to `working_directory`. |
| `no_git` | `false` | Set `true` for filesystem-only scan (`--no-git`). |
| `fail_on_findings` | `true` | If `true`, exit 1 when `secret_count > 0`. |

## Outputs

| Output | Description |
|--------|-------------|
| `secret_count` | Number of secret findings in the JSON report. |
| `scan_status` | `clean`, `findings_found`, or `scanner_error`. |

## Example

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0

- uses: thadiust/secrets-gitleaks@main
  with:
    working_directory: "."
    fail_on_findings: true
```

## Opinion on “block on one finding?”

For learning and **fail-closed** CI, **yes**: one real leak is enough to block until triaged. Tune noise later with `.gitleaksignore` / config, not by weakening the default in the action.
