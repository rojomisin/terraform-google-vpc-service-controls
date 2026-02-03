# Upgrading to v8.0

The v8.0 release contains breaking changes to input variable types. All list-based inputs for resources and policies have been changed to maps to prevent unnecessary resource recreations when modifying policy rules.

## Migration Instructions

### Resources Variables Changed from List to Map

**Affected modules:**
- `modules/regular_service_perimeter`
- `modules/bridge_service_perimeter`

**Changed variables:**
- `resources` - now requires `map(string)` instead of `list(string)`
- `resources_dry_run` - now requires `map(string)` instead of `list(string)`

**Before v8.0:**
```hcl
module "regular_service_perimeter_1" {
  source = "terraform-google-modules/vpc-service-controls/google//modules/regular_service_perimeter"

  resources = [
    "projects/123456789",
    "projects/987654321"
  ]

  resources_dry_run = [
    "projects/123456789"
  ]
}
```

**After v8.0:**
```hcl
module "regular_service_perimeter_1" {
  source = "terraform-google-modules/vpc-service-controls/google//modules/regular_service_perimeter"

  resources = {
    project1 = "projects/123456789"
    project2 = "projects/987654321"
  }

  resources_dry_run = {
    project1 = "projects/123456789"
  }
}
```

### Policy Variables Changed from List to Map

**Affected modules:**
- `modules/regular_service_perimeter`

**Changed variables:**
- `ingress_policies` - now requires `map(object(...))` instead of `list(object(...))`
- `egress_policies` - now requires `map(object(...))` instead of `list(object(...))`
- `ingress_policies_dry_run` - now requires `map(object(...))` instead of `list(object(...))`
- `egress_policies_dry_run` - now requires `map(object(...))` instead of `list(object(...))`

**Removed variables** (no longer needed with map inputs):
- `ingress_policies_keys`
- `egress_policies_keys`
- `ingress_policies_keys_dry_run`
- `egress_policies_keys_dry_run`
- `resource_keys`
- `resource_keys_dry_run`

**Before v8.0:**
```hcl
module "regular_service_perimeter_1" {
  source = "terraform-google-modules/vpc-service-controls/google//modules/regular_service_perimeter"

  ingress_policies = [
    {
      title = "Allow Access from everywhere"
      from = {
        identities = ["user:admin@example.com"]
        sources = {
          access_levels = ["*"]
        }
      }
      to = {
        resources = ["*"]
        operations = {
          "storage.googleapis.com" = {
            methods = ["google.storage.objects.get"]
          }
        }
      }
    },
    {
      title = "Allow Access from project"
      from = {
        sources = {
          resources = ["projects/123456"]
        }
        identity_type = "ANY_SERVICE_ACCOUNT"
      }
      to = {
        resources = ["*"]
        operations = {
          "storage.googleapis.com" = {
            methods = ["google.storage.objects.get"]
          }
        }
      }
    }
  ]

  # Optional: use custom keys to prevent shuffling
  ingress_policies_keys = [
    "allow_everywhere",
    "allow_from_project"
  ]
}
```

**After v8.0:**
```hcl
module "regular_service_perimeter_1" {
  source = "terraform-google-modules/vpc-service-controls/google//modules/regular_service_perimeter"

  ingress_policies = {
    allow_everywhere = {
      title = "Allow Access from everywhere"
      from = {
        identities = ["user:admin@example.com"]
        sources = {
          access_levels = ["*"]
        }
      }
      to = {
        resources = ["*"]
        operations = {
          "storage.googleapis.com" = {
            methods = ["google.storage.objects.get"]
          }
        }
      }
    }
    allow_from_project = {
      title = "Allow Access from project"
      from = {
        sources = {
          resources = ["projects/123456"]
        }
        identity_type = "ANY_SERVICE_ACCOUNT"
      }
      to = {
        resources = ["*"]
        operations = {
          "storage.googleapis.com" = {
            methods = ["google.storage.objects.get"]
          }
        }
      }
    }
  }

  # No longer need ingress_policies_keys - keys are in the map!
}
```

## Benefits of This Change

### Prevents Unnecessary Resource Replacements

**Before v8.0:** Adding, removing, or reordering policies in the middle of a list would cause Terraform to replace multiple unrelated resources due to index shifting.

Example of the problem:
```
# Adding a new policy at index 2 would cause:
ingress_policies["2"] -> ingress_policies["3"]  # forces replacement
ingress_policies["3"] -> ingress_policies["4"]  # forces replacement
ingress_policies["4"] -> ingress_policies["5"]  # forces replacement
# ... all subsequent policies get recreated
```

**After v8.0:** Changes only affect the specific policy being modified.

Example with map keys:
```
# Adding a new policy only creates one resource:
ingress_policies["new_rule"] will be created
# Existing policies remain untouched
```

### Improved Code Readability

Map keys provide semantic meaning:
```hcl
ingress_policies = {
  allow_engineers_from_tailscale_bigquery = { ... }
  allow_infra_ingress_unauthed_artifact_registry = { ... }
  allow_dev_stg_to_prd_artifact_registry = { ... }
}
```

Instead of numeric indices:
```hcl
ingress_policies[0], ingress_policies[1], ingress_policies[2]  # What do these represent?
```

## State Migration

When upgrading, Terraform will plan to **replace all resources** because the keys have changed. You have two options:

### Option 1: Accept Replacement (Safest)

Let Terraform recreate the resources with new keys. This is safe because VPC Service Controls policies are declarative.

```bash
terraform plan   # Review the changes
terraform apply  # Accept the replacements
```

### Option 2: State Migration (Advanced)

Migrate your existing state to use the new map keys to avoid replacements.

**Example for resources:**
```bash
# If you had: resources = ["projects/123", "projects/456"]
# And want:    resources = { project1 = "projects/123", project2 = "projects/456" }

terraform state mv \
  'module.perimeter.google_access_context_manager_service_perimeter_resource.service_perimeter_resource["0"]' \
  'module.perimeter.google_access_context_manager_service_perimeter_resource.service_perimeter_resource["project1"]'

terraform state mv \
  'module.perimeter.google_access_context_manager_service_perimeter_resource.service_perimeter_resource["1"]' \
  'module.perimeter.google_access_context_manager_service_perimeter_resource.service_perimeter_resource["project2"]'
```

**Example for policies:**
```bash
# If you had: ingress_policies_keys = ["allow_everywhere", "allow_from_project"]
# The state keys were: ["0"], ["1"]
# You want map keys: ["allow_everywhere"], ["allow_from_project"]

terraform state mv \
  'module.perimeter.google_access_context_manager_service_perimeter_ingress_policy.ingress_policies["0"]' \
  'module.perimeter.google_access_context_manager_service_perimeter_ingress_policy.ingress_policies["allow_everywhere"]'

terraform state mv \
  'module.perimeter.google_access_context_manager_service_perimeter_ingress_policy.ingress_policies["1"]' \
  'module.perimeter.google_access_context_manager_service_perimeter_ingress_policy.ingress_policies["allow_from_project"]'
```

⚠️ **Note:** State migration is error-prone. Always backup your state file before attempting manual state moves.

## Dynamic Resources

If you're generating resources dynamically from data sources, convert the list to a map:

**Before v8.0:**
```hcl
data "google_projects" "in_folder" {
  filter = "parent.id:${var.folder_id}"
}

locals {
  project_numbers = [for p in data.google_projects.in_folder.projects : p.number]
}

module "perimeter" {
  source = "..."
  resources = local.project_numbers
}
```

**After v8.0:**
```hcl
data "google_projects" "in_folder" {
  filter = "parent.id:${var.folder_id}"
}

locals {
  project_numbers = [for p in data.google_projects.in_folder.projects : p.number]
  resources_map = { for idx, num in local.project_numbers : "project_${idx}" => num }
}

module "perimeter" {
  source = "..."
  resources = local.resources_map
}
```

Or use project IDs for better semantics:
```hcl
locals {
  resources_map = { for p in data.google_projects.in_folder.projects : p.project_id => p.number }
}

module "perimeter" {
  source = "..."
  resources = local.resources_map
}
```
