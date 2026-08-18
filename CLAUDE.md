# WAM CN — Universal Terraform Module Conventions

These conventions apply to every `terraform-azure-*` module in the WAM CN ecosystem.
This file lives at `~/.claude/CLAUDE.md` and is inherited by all repos.

---

## 1. Resource keys

- Primary resource always uses key `"this"`
- Secondary resources use a semantic qualifier: `.admins`, `.readers`, `.tls`, `.backup`
- One resource of a given type per module → always `"this"`, never `"default"` or a list sentinel

---

## 2. File layout

Mandatory files at module root:

```
main.tf
variables.tf
outputs.tf
terraform.tf
README.md          # auto-generated via terraform-docs
CHANGELOG.md       # auto-generated via release-please
Makefile
GOALS.md
TESTING.md
CONTRIBUTING.md
CODE_OF_CONDUCT.md
SECURITY.md
LICENSE
.gitignore
examples/
tests/
.github/
```

- `locals.tf` only when flattening logic exceeds ~50 lines
- `modules/` only when a technical blocker requires a sub-module split (see rule 18)
- Section headers in `main.tf`: lowercase, single line, one per resource type

```hcl
# keyvault
# role assignments
# private endpoint
# keys
```

---

## 3. `terraform.tf` versioning

```hcl
terraform {
  required_version = "~> 1.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}
```

- Always pessimistic constraint `~>`, never `>= x.y.z`
- Only include providers the module actually uses

---

## 4. Provider

- Always use `azurerm` (HashiCorp) as the default provider
- Only use `azapi` as a last resort — when a resource or property is not yet available in `azurerm` and there is a concrete business need that cannot wait for the provider to catch up
- Document the reason in a comment whenever `azapi` is used
- Reason to prefer `azurerm`: AzApi is a thin REST wrapper that produces noisy TF plans, is less stable, and largely defeats the declarative purpose of IaC

---

## 5. Primary config variable

- Use a service-specific noun: `var.vault`, `var.storage`, `var.cluster`, `var.vnet`
- Never use `var.config`, `var.instance`, or `var.environment`
- Do not use `var.naming` — the caller handles naming before invoking the module. The name is passed directly on the primary object: `var.vault.name`

---

## 6. Four outer globals

These are the only variables allowed to have defaults:

```hcl
var.location
var.resource_group_name
var.tags
```

- Inner object value always wins over outer global
- Use `coalesce(lookup(var.vault, "location", null), var.location)` to resolve

---

## 7. Type definitions

- All inputs strictly typed — never `type = any`
- No deviation defaults — no `try()`-for-defaulting in `main.tf`. Provider default applies unless the caller sets the field, or a default is declared on the type in `variables.tf`
- Never `optional(bool, true)` — silently sets production flags invisible to callers
- Security-relevant flags must always be explicit: `purge_protection_enabled`, `public_network_access_enabled` — no default argument, ever

**Setting defaults**

When a module wants to nudge a property away from the raw provider default (not security-relevant, just a sane fallback), declare the default on the type itself in `variables.tf`:

```hcl
account_replication_type = optional(string, "GRS")
```

`main.tf` then references `var.storage.account_replication_type` directly — no `try()`, no fallback logic in the resource block. `variables.tf` is the one place a deviation from the provider default is declared, so callers can read the contract in one spot instead of hunting through `main.tf`.

Use this pattern for example:
- Replication types (`"GRS"`, `"ZRS"`)
- Retention periods
- Any property where nudging the default is a convenience, not a compliance-critical guarantee

Do not give a default to properties where the wrong value could silently misconfigure a resource, or to anything security-relevant — those stay `optional(type)` with no default, and required/explicit at the call site.

---

## 8. `lookup()` over `try()`

- Use `lookup(obj, "key", null)` for optional properties on objects
- `try()` swallows unknown-at-plan-time values from other resources — masks real bugs and causes plan/apply divergence
- Only use `try()` for deep attribute chains where `lookup()` cannot reach

---

## 9. `for_each` only — never `count`

- `count` loses resource identity on list mutations
- `for_each` is stable across map key changes
- Key normalisation when Azure forbids underscores: `replace(each.key, "_", "-")` (e.g. KV secret names)

---

## 10. Dynamic blocks — one canonical form

Zero-or-one:
```hcl
for_each = lookup(var.vault, "key", null) != null ? { "this" = lookup(var.vault, "key", null) } : {}
```

Zero-or-many:
```hcl
for_each = lookup(each.value, "items", {})
```

- The `for_each` key is always `"this"` for zero-or-one blocks — never `"default"` or any other sentinel

---

## 11. Validation — mandatory

Every module must validate:

- `location` is provided either on the primary object or via the outer global
- `resource_group_name` same rule
- Enum validations where the value domain is small (e.g. `create_mode`, SKU names)

---

## 12. `lifecycle` — use sparingly

- `ignore_changes`: only when a property is managed out-of-band. Always document the reason in a comment.
  - Allowed examples: `expiration_date` on KV keys (rotation policy), `route` on route tables (`azurerm_route` manages it)
- `prevent_destroy` / `create_before_destroy`: require a documented reason in a comment
- Never use `depends_on` when Terraform can infer order from the graph — it causes unnecessary resource recreation on unrelated changes

---

## 13. Outputs

- Export the full primary resource as `output "vault"` (or equivalent service noun)
- Mark TLS private keys, passwords, connection strings, and access keys as `sensitive = true`
- Aggregate resource outputs are exported raw — some fields are sensitive but not the whole object

---

## 14. Role assignments — bundle implicit ones

Bundle role assignments when they are required for Terraform itself to function, or for the resource's primary runtime purpose. Two canonical examples:

- **Key Vault Administrator** on the Terraform service principal — without it the second apply fails; Terraform cannot manage secrets, keys, or certs
- **AcrPull on the UAI** for Container App modules — without it the container fails to start at runtime

General rule:
- If a role assignment is an implicit enabler → bundle it in the module, default on
- If a role assignment spans two separate modules and is not an implicit enabler → leave it to the root caller

---

## 15. AVM-aligned embedded interfaces

Every subject module exposes these five interfaces:

```
private_endpoints       # map of objects — plural, never singular
managed_identities      # single object
customer_managed_key    # single object
role_assignments        # map of objects
diagnostic_settings     # map of objects
```

- `private_endpoints` is always a plural map — one subject may have multiple PEs
- `managed_identities` accepts external UAI IDs as input — the module never creates UAIs internally

---

## 16. Private endpoint — dual pattern

Both embedded and standalone are valid. The choice belongs to the caller.

**Use embedded when:**
- `public_network_access_enabled = false` and the module manages data-plane children (secrets, keys, certs) in the same apply — Terraform has no network path without it (ref: azurerm provider issue #23724, open)
- The PE and the subject resource live in the same state file and workload

**Use standalone (`terraform-azure-pe`) when:**
- The PE lives in a different state file (e.g. platform team owns networking, app team owns the resource)
- The subject resource has public access enabled and the PE is optional or additive
- You are composing across multiple resources in a pattern module

The embedded interface shape is always `private_endpoints` — plural map of objects, never a singular object.

---

## 17. UAI — always standalone, never embedded

- Never create `azurerm_user_assigned_identity` inside a subject module
- Accept identity resource IDs as plain string inputs
- Use `terraform-azure-uai` as the source
- Reason: embedding causes complex conditional logic when multiple UAIs are needed for different purposes (e.g. a Container App needing one UAI for KV access and a separate one for ACR pull)

---

## 18. Sub-modules — only for technical blockers

Only create a `modules/<name>/` sub-module when one of these four blockers applies:

| # | Blocker | Description | Example |
|---|---|---|---|
| S1 | `for_each` on unknown keys | Keys not known at plan time | Sub-module accepts computed values as static string inputs from the caller |
| S2 | A↔B resource ID cycle | A needs B's ID, B needs A's ID | `terraform-azure-fwp`: policy → ip-groups → rule-groups |
| S3 | Two-phase Azure API provisioning | Parent must be fully provisioned before child attaches | `terraform-azure-vwan`: use data source lookup in sub-module, not direct resource reference — avoids `depends_on` and the recreation bug it causes |
| S4 | Independent operational lifecycle | Different team or cadence owns the child resources | `terraform-azure-aa`: automation account (infrastructure) vs runbooks (application content) |

**Never** split for neatness, "just in case", or pure containment. KV→secrets and SA→containers are bundled, not split.

---

## 19. Greenfield vs brownfield — `use_existing_*`

- Modules that can be invoked in both modes support `use_existing_*` flags
- The module accepts either a string reference to an existing resource or creates inline
- Example: platform team creates the KV, app team manages secrets in it via a second invocation of the same module pointing at the existing vault

---

## 20. `.gitignore` policy

- Module root: **track** `.terraform.lock.hcl` (ensures reproducibility for consumers)
- `examples/*`: **ignore** it (each example initialises its own)

---

## 21. `examples/` structure

Standard subfolders: `default/`, `complete/`, and one per major optional feature (`private-endpoint/`, `keys/`, `secrets/`, `certs/`, `access-policies/`, `network-acls/`).

Each example contains: `main.tf`, `terraform.tf`, `naming.tf`, `README.md`

Minimal call shape:

```hcl
module "kv" {
  source = "../../"

  vault = {
    name                = module.naming.key_vault.name_unique
    location            = module.rg.groups.demo.location
    resource_group_name = module.rg.groups.demo.name
  }

  tags = { environment = "demo" }
}
```

---

## 22. Testing

- Go-based Terratest in `tests/`
- `deploy_test.go` deploys each example, asserts key properties, destroys
- Tests exercise the module through its examples — they do not re-implement module logic
- Every example must have test coverage
- Every release runs the full suite via `make test`

---

## 23. Makefile — mandatory targets

```
install-tools
validate
fmt
docs
test
test-parallel
all
```

---

## 24. README

Auto-generated via terraform-docs between markers. Never hand-edit between them. Run `make docs` to regenerate.

```markdown
<!-- BEGIN_TF_DOCS -->
...
<!-- END_TF_DOCS -->
```

---

## 25. CHANGELOG

Driven by release-please from Conventional Commits. Never hand-write entries.

---

## 26. `moved` blocks — major version refactors

When a major version bump renames or restructures resources, capture all `moved` blocks in a dedicated versioned file at the module root:

```
moved_v4_to_v5.tf
moved_v5_to_v6.tf
```

- One file per major version transition — never append to a previous version's file
- Do **not** put `moved` blocks inline in `main.tf`
- Every resource rename, key change (`resource.foo` → `resource.this`), or collection conversion (`resource.this` → `resource.this["this"]`) that would otherwise cause a destroy/recreate gets a `moved` block here

Example (`moved_v4_to_v5.tf`):

```hcl
moved {
  from = azurerm_key_vault.keyvault
  to   = azurerm_key_vault.this["this"]
}

moved {
  from = azurerm_key_vault_secret.secrets
  to   = azurerm_key_vault_secret.this
}
```

`moved` blocks inside a reusable module work correctly for all callers on upgrade — Terraform resolves them relative to each module instance's address. Source: https://developer.hashicorp.com/terraform/language/modules/develop/refactoring

- `feat:` → minor bump
- `fix:` → patch bump
- `feat!:` → major bump (breaking change)
