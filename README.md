# azure-oidc-github-deploy

![ci](https://github.com/Supp3r-Man/azure-oidc-github-deploy/actions/workflows/ci.yml/badge.svg)

**Keyless deployment from GitHub Actions to Azure, with the trust model written down and tested.** No client secret and no stored credential: a workflow's short-lived OIDC token is exchanged for an Entra access token through a *federated identity credential*. The Terraform creates two identities (a read-only one for pull requests, a Contributor on one resource group for the protected `production` environment), and the Python in `entra_guard/` answers the two questions that matter: **which tokens does each application accept, and what can they then do?**

> **What is verified, and what is not.** 121 tests pass locally on Python 3.13, `python -m entra_guard` runs, and `terraform validate` passes against the **real `azuread` ~> 3 and `azurerm` ~> 4 provider schemas**. I changed the code on purpose nine times (case-insensitive subject, ignored audience or issuer, a scope prefix with no segment boundary, ignored `notActions`, Contributor allowed to grant access, and so on) and tests failed each time. **No Azure subscription or Entra tenant is involved.** The exchange and RBAC logic is *my model of documented behaviour*, never compared with a real tenant, and `deploy.yml` was never run. GitHub Actions on Linux passed the 121 tests and `python -m entra_guard` on Python 3.11 and 3.13, `terraform fmt`/`init`/`validate` against the real providers, and `actionlint` (2026-10-17).

## The trust model

```
GitHub job ──OIDC token (iss, sub, aud)──▶ the application named by client-id
                                              │ one of its federated credentials must match all three, exactly
                                              ▼
                                    service principal ──RBAC──▶ resources in scope
```

| Identity | Federated credential subject | Role |
|---|---|---|
| `plan` | `repo:ORG/REPO:pull_request` | Reader on the resource group |
| `deploy` | `repo:ORG/REPO:environment:production` | Contributor on the resource group |

The Terraform is in `terraform/`; the deploy workflow passes the **client id** of the right application (an identifier, not a secret) and Entra does the rest.

## What the model does (`python -m entra_guard`)

```
audit: no findings

  token                                plan app     deploy app
  pull request in the repo             read         denied
  job in the production environment    denied       read, write
  job in the staging environment       denied       denied
  push to main (no environment)        denied       denied
  fork with the same repo name         denied       denied
  another repo in the same org         denied       denied
  right subject, wrong audience        denied       denied
```

- **`exchange`**: a token is accepted only if its issuer, subject and audience each equal one credential's values, case-sensitively. A wildcard in a credential is just text. A missing or wrongly typed claim is refused.
- **`rbac`**: assignments apply to their scope and everything below it (compared by whole path segment, so `rg-payments` does not contain `rg-payments2`), permissions from all assignments add up, a role grants `actions` minus `notActions` and, separately, `dataActions`. So Contributor can write a storage account but cannot grant access *and* cannot read blob data.
- **`audit`** (11 rules over the Terraform): `FIC001` wildcard subject, `FIC002` wrong issuer, `FIC003` unusual audience (warn), `FIC004` subject outside this repository, `FIC005` branch subject on a writer (warn), `RBAC001` Owner or User Access Administrator, `RBAC002` broad role wider than a resource group, `RBAC003` assignment at the root or a management group (warn), `RBAC004` a pull-request identity that can change things, `APP001` a client secret.

The tests run **every token shape (12) against both applications of the real Terraform** and require that none is accepted by both.

## Why exact subjects and an environment

A pull-request token can be minted by anyone who can open a PR that edits a workflow, so the PR identity is read-only (`RBAC004` enforces it). The deploy identity trusts one *environment*, not a branch: a branch subject lets anyone who can push that branch deploy, whereas an environment's required reviewers are a gate GitHub enforces before the token is issued (`FIC005` warns on the branch form).

## Things I got wrong or checked

- My first audit expectations for a whole-org subject (`repo:acme/*`) listed only the wildcard finding; it also trusts every other repository in the organisation, so `FIC004` fires too. The test now says so.
- One mutation (deleting the `Authorization/*/Write` exclusion from Contributor) did not apply the first time because my `sed` pattern missed it; I redid it and it is caught. A mutation that changes nothing proves nothing, so the check script reports "no change".
- `terraform validate` needed `subscription_id` in the `azurerm` provider block (azurerm 4); the reference configuration sets it from a variable.

## Honest limits

- **A model, not Azure.** I could not run it against a tenant, so every behaviour here (exact matching, case sensitivity, RBAC arithmetic) is taken from documentation and should be rechecked against it. Entra's *flexible* federated credentials, which can match claims with an expression, are **not modelled**; a wildcard-like expression there would be a different (and broader) trust than anything tested here.
- **The role table is abridged.** `Reader`, `Contributor`, `Owner`, `User Access Administrator` and `Storage Blob Data Contributor` are included with only the permissions this repository relies on. Real built-in roles have more `notActions` (and change over time).
- **Not modelled:** deny assignments, assignment conditions, PIM or eligible assignments, Conditional Access for workload identities, tenant restrictions, and the limit on federated credentials per application.
- The audit reads Terraform text: a value that comes from a module, `for_each` or a data source stays an unresolved `<ref>` and cannot be judged (a test pins that it stays visible).
- GitHub-side protections (the `production` environment's reviewers, branch protection) are **not in code here**; the trust model depends on them being configured.
- `deploy.yml` (and the `az containerapp` commands in it) is a reference pipeline, never run.

## Run it

```
python -m pip install pytest pyyaml python-hcl2
python -m pytest
python -m entra_guard                          # audit + who-can-do-what table
python -m entra_guard terraform --org myorg --repo myrepo
terraform -chdir=terraform init -backend=false && terraform -chdir=terraform validate
```
