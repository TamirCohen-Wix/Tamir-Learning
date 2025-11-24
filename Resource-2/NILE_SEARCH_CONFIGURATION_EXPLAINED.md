# Nile Search Configuration Explained

## Overview

This document explains the `prime_app` search section configuration options for Nile Search integration. This configuration is typically found in your `BUILD.bazel` file.

---

## Configuration Options

### 1. **`pii`** (Boolean)

**What it is:**
- Determines whether to use the **PII (Personally Identifiable Information) cluster**
- Controls which Elasticsearch cluster your search index will be stored in

**When to use:**
- Set to `True` if you:
  - Are currently indexing PII fields (personal data like names, emails, etc.)
  - Plan to index PII fields in the near future
- Set to `False` if you:
  - Don't have any PII fields
  - Won't need PII fields in the future

**Why it matters:**
- PII data requires special handling for GDPR compliance
- PII clusters have additional security and encryption
- Separate clusters ensure proper data isolation

**Example:**
```python
search = {
    "pii": True,  # Use PII cluster
    # ... other options
}
```

---

### 2. **`tenancy`** (String)

**What it is:**
- Specifies the **tenancy type** of your API
- Must match the tenancy type configured in your SDL (Simple Data Layer)

**Available options:**
- `"Instance"` - Each instance has its own data
- `"Metasite"` - Data is scoped to a metasite (most common for Wix apps)
- `"TargetAccount"` - Data is scoped to a target account
- `"Single"` - Single tenant (no multi-tenancy)

**Why it matters:**
- Ensures search queries are properly scoped to the correct tenant
- Prevents data leakage between tenants
- Must match SDL configuration or search won't work correctly

**In your project:**
Based on your SDL configuration:
```scala
.withTenantExtractor(TenantExtractors.metaSite)
```
You should use: `"tenancy": "Metasite"`

**Example:**
```python
search = {
    "tenancy": "Metasite",  # Must match SDL tenant extractor
    # ... other options
}
```

---

### 3. **`schemas.{label}`** (Schema Label)

**What it is:**
- A **schema label** that identifies a specific search schema version
- Allows you to have multiple schemas (useful for migrations)

**Why multiple schemas?**
- **Migrations**: When you need to change the search index structure
  - Keep old schema for backward compatibility
  - Create new schema with updated structure
  - Migrate data gradually
- **A/B Testing**: Test different index structures
- **Versioning**: Maintain multiple versions of your search schema

**Example:**
```python
search = {
    "schemas": {
        "v1": {  # Schema label
            # ... schema configuration
        },
        "v2": {  # Another schema label (for migration)
            # ... schema configuration
        }
    }
}
```

**Common labels:**
- `"default"` - Default/current schema
- `"v1"`, `"v2"` - Versioned schemas
- `"migration"` - Temporary schema for migrations

---

### 4. **`schemas.{label}.{cluster}`** (Elasticsearch Cluster)

**What it is:**
- Specifies which **Elasticsearch cluster** to use for this schema
- Different clusters for different environments (POC vs Production)

**For POC (Proof of Concept) / Development:**
- **Non-PII data**: `"es-poc"`
- **PII data**: `"es-poc-pii"`

**For Production:**
- **You must ask in `#nile-search` channel** for the cluster name
- Cluster names are environment-specific
- Different clusters for different regions/environments

**Why different clusters?**
- **Isolation**: Separate dev/test/prod environments
- **Performance**: Production clusters are optimized
- **Security**: PII clusters have additional security measures
- **Compliance**: PII data must be in compliant clusters

**Example:**
```python
search = {
    "schemas": {
        "default": {
            "cluster": "es-poc",  # For POC/development
            # OR
            "cluster": "es-bookings-pii",  # For production (from your BUILD.bazel)
        }
    }
}
```

---

## Complete Configuration Example

### Your Current Configuration (from BUILD.bazel)

```python
prime_app(
    # ... other config
    search = {
        "tenancy": "Metasite",  # Matches your SDL TenantExtractors.metaSite
        "schemas": {
            "default": {  # Schema label
                "cluster": "es-bookings-pii",  # Production PII cluster
                "pii": True,  # Using PII cluster
            }
        }
    }
)
```

### POC/Development Configuration

```python
search = {
    "tenancy": "Metasite",
    "schemas": {
        "default": {
            "cluster": "es-poc",  # POC cluster for non-PII
            "pii": False,
        }
    }
}
```

### Production with PII Configuration

```python
search = {
    "tenancy": "Metasite",
    "schemas": {
        "default": {
            "cluster": "es-bookings-pii",  # Production PII cluster (from #nile-search)
            "pii": True,  # Using PII cluster
        }
    }
}
```

### Migration Example (Multiple Schemas)

```python
search = {
    "tenancy": "Metasite",
    "schemas": {
        "v1": {  # Old schema
            "cluster": "es-bookings-pii",
            "pii": True,
        },
        "v2": {  # New schema (migration)
            "cluster": "es-bookings-pii",
            "pii": True,
        }
    }
}
```

---

## How This Relates to Your constraint_filter Issue

### Configuration Impact

The `prime_app` search configuration determines:
1. **Which Elasticsearch cluster** receives your search queries
2. **How tenancy is handled** (affects how constraint_filter is scoped)
3. **Whether PII handling** is enabled (affects security/encryption)

### Connection to constraint_filter

The constraint_filter is generated by Nile Search infrastructure based on:
- Your `tenancy` configuration (affects how location restrictions are applied)
- The schema structure (affects field path format)
- The cluster configuration (where the search index lives)

**However**, the constraint_filter field path issue is **NOT** directly related to this configuration. The issue is about:
- Field path translation (Domain → WQL)
- Not about cluster or tenancy configuration

---

## Configuration Checklist

When setting up Nile Search, ensure:

- [ ] **`tenancy`** matches your SDL configuration
  - Check: `TenantExtractors.metaSite` → `"tenancy": "Metasite"`
  
- [ ] **`pii`** is set correctly
  - `True` if you have/will have PII fields
  - `False` if no PII fields
  
- [ ] **`cluster`** is appropriate for your environment
  - POC: `"es-poc"` or `"es-poc-pii"`
  - Production: Ask in `#nile-search` channel
  
- [ ] **`schemas.{label}`** is meaningful
  - Use `"default"` for single schema
  - Use versioned labels (`"v1"`, `"v2"`) for migrations

---

## Getting Help

**For Production Cluster Names:**
- Ask in `#nile-search` Slack channel
- Provide: App name, environment, PII requirements

**For Configuration Issues:**
- Check that `tenancy` matches SDL configuration
- Verify cluster names are correct for your environment
- Ensure PII flag matches your data requirements

---

## Summary

| Option | Purpose | Your Value |
|--------|---------|------------|
| `pii` | Use PII cluster? | `True` (you have PII) |
| `tenancy` | Tenancy type | `"Metasite"` (matches SDL) |
| `schemas.{label}` | Schema identifier | `"default"` |
| `schemas.{label}.cluster` | Elasticsearch cluster | `"es-bookings-pii"` (production) |

**Key Points:**
- `tenancy` must match SDL configuration
- `pii` depends on whether you index PII fields
- `cluster` differs for POC vs Production
- Multiple schemas useful for migrations

