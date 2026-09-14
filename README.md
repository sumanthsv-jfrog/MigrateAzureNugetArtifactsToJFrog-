# Azure Artifacts → JFrog Artifactory Migration (NuGet)

A collection of Azure DevOps pipelines to migrate **NuGet** packages from Azure Artifacts to JFrog Artifactory Cloud, with delta migration support to avoid re-processing already-migrated packages.

---

## Migration Flow

```
Azure Artifacts NuGet feed
        ↓
   ADO Pipeline
        ↓
JFrog Artifactory
```

---

## Two Migration Methods

### Option 1 — Direct Download & Upload

Downloads `.nupkg` files directly from Azure Artifacts feed using the REST API, then uploads to JFrog Artifactory local repository.

```
Azure Artifacts NuGet feed
        ↓  curl -L (follows 303 redirect to blob storage)
   ADO Pipeline
        ↓  jf rt upload
JFrog Local Repository
```

**Pros:**
- Simple setup — no extra repos needed in JFrog
- Works independently of JFrog remote repo configuration
- Full control over which packages are migrated
- Supports delta migration (skips already-migrated packages)

**Cons:**
- Must follow 303 redirects (`-L` flag required)
- Downloads happen on the ADO agent

---

### Option 2 — Pull via JFrog Remote Repo → Copy to Local

Creates a JFrog Remote Repository pointing to Azure Artifacts NuGet feed. A `.csproj` is generated dynamically from the package list, then `jf dotnet restore` pulls packages through JFrog (auto-cached), and finally copies them to a local repository.

```
Azure Artifacts NuGet feed
        ↓  JFrog Remote Repo proxies & caches
JFrog Remote Repository (cache)
        ↓  jf rt cp
JFrog Local Repository
```

**Pros:**
- JFrog handles redirects, auth, and metadata natively
- Packages properly indexed by JFrog
- No redirect handling needed

**Cons:**
- Requires creating a Remote Repo in JFrog pointing to Azure feed
- PAT must be stored in JFrog remote repo configuration
- No built-in delta tracking (relies on JFrog's on-demand caching instead)

---

## Repository Structure

```
azure-to-jfrog-nuget-migration/
  README.md
  nuget-packages-to-sync.txt     # Curated NuGet package list for Pipeline 2
  migrated-packages.txt          # Auto-maintained tracking file (created by pipelines)
  pipelines/
    1-list-packages.yml          # List all packages in Azure NuGet feed
    2-migrate-specific.yml       # Option 1: Migrate specific packages from list (delta-aware)
    3-migrate-all.yml            # Option 1: Migrate all packages in feed (delta-aware)
    4-migrate-via-remote.yml     # Option 2: Pull via JFrog remote repo
```

---

## Prerequisites

### Azure DevOps
- Azure Artifacts NuGet feed with packages present
- Personal Access Token (PAT) with **Packaging Read** scope
- Pipeline variable `ADO_PAT` set as secret
- JFrog Azure DevOps Extension installed from Marketplace
- Self-hosted agent registered in `Default` pool with `jq` and `curl` installed
- **Contribute** permission granted to the Build Service identity on the repo (needed for Pipelines 2 & 3 to commit `migrated-packages.txt`)

### JFrog Artifactory
- JFrog Cloud instance
- Local NuGet repository created
- Service connection configured in ADO Project Settings
- **Option 2 only:** Remote NuGet repository pointing to Azure Artifacts feed

### Build Agent (Option 2 only)
- .NET SDK installed (`dotnet --version`)

---

## Pipeline Variables

| Variable | Description | Secret |
|---|---|---|
| `ADO_PAT` | Azure DevOps PAT with Packaging Read scope | ✅ Yes |

> JFrog authentication is handled by the service connection — no JFrog credentials needed as pipeline variables.

---

## Grant Repo Contribute Permission (Required for Delta Tracking)

Pipelines 2 and 3 commit updates to `migrated-packages.txt` back to the repo. This requires the Build Service identity to have write access:

1. Go to **Project Settings → Repositories**
2. Select your repository
3. Click **Security** tab
4. Search for `{project} Build Service ({org})`
5. Set **Contribute** to **Allow**

Also ensure `checkout: self` includes `persistCredentials: true` in the pipeline (already included in the YAML below).


## Delta Migration — How It Works

A shared tracking file, **`migrated-packages.txt`**, records every package (`packageId:version`) that has already been migrated to JFrog. Both Pipeline 2 and Pipeline 3 read and update this same file, so they stay in sync no matter which one runs first or how often.

```
migrated-packages.txt
├── Microsoft.Extensions.DependencyInjection:10.0.11
├── Newtonsoft.Json:13.0.4
└── Serilog:4.4.0
```

### First run (file doesn't exist)
All requested/available packages are treated as new → migrated → file created.

### Subsequent runs
Only packages **not already listed** in `migrated-packages.txt` are downloaded and uploaded. If nothing is new, the pipeline:
- Skips the download step
- Skips the upload step (`jf rt u` task is conditioned out — shown as *skipped* in the pipeline UI, not just a no-op)
- Skips the git commit/push

This makes both pipelines **idempotent** — running them repeatedly with unchanged input does nothing extra.

```
Day 1: Pipeline 2 migrates 5 specific packages    → migrated-packages.txt = [5]
Day 2: New packages land in Azure feed
Day 2: Pipeline 3 (delta) runs                    → only new packages processed, existing 5 skipped
Day 3: Pipeline 2 runs again with same list       → 0 to migrate → fully skipped
```

---

## Pipeline 1 — List Packages

Fetches all NuGet packages and all versions from Azure feed with pagination support (up to 100,000 packages). Publishes a `nuget-packages-to-sync.txt` file as a pipeline artifact.

**Output format:** `packageId:version`

```
Microsoft.Extensions.DependencyInjection:10.0.11
Microsoft.Extensions.Logging:10.0.11
Newtonsoft.Json:13.0.4
Serilog:4.4.0
```

**When to use:** To see what's in your Azure feed, or to generate an updated list before a selective sync.

---

## Pipeline 2 — Migrate Specific Packages (Option 1, Delta-aware)

Reads `nuget-packages-to-sync.txt` from the repo root and migrates only the packages **not already present** in `migrated-packages.txt`.

**nuget-packages-to-sync.txt format:**
```
# Microsoft Extensions
Microsoft.Extensions.DependencyInjection:10.0.11
Microsoft.Extensions.Logging:10.0.11

# Third party
Newtonsoft.Json:13.0.4
Serilog:4.4.0
```

**When to use:** When you want full control over which packages get migrated.

**Steps:**
1. Clean the upload working directory
2. Compare `nuget-packages-to-sync.txt` against `migrated-packages.txt` → compute what's actually new
3. Download only the new `.nupkg` files (follows 303 redirect with `-L`)
4. Skip upload/commit entirely if nothing new was found
5. Upload to JFrog local repository
6. Update and commit `migrated-packages.txt`

---

## Pipeline 3 — Migrate All Packages (Option 1, Delta-aware)

Fetches **every** package currently in the Azure NuGet feed, computes the delta against `migrated-packages.txt`, and migrates only what's new. Uses pagination (100 per page) to handle large feeds.

**When to use:** For a full one-time migration, or as a recurring job to pick up newly added packages in the feed.

**Steps:**
1. Clean the upload working directory
2. Paginate through all packages currently in the Azure feed
3. Compute delta: `comm -23 current-packages.txt migrated-packages.txt`
4. Download only the delta packages (follows 303 redirect)
5. Skip upload/commit entirely if delta is empty
6. Upload to JFrog local repository
7. Update and commit `migrated-packages.txt`

---

## Pipeline 4 — Migrate via Remote Repo (Option 2)

**Setup required:** Create a Remote Repository in JFrog pointing to your Azure NuGet feed.

### JFrog Remote Repo Setup

| Field | Value |
|---|---|
| Repository Key | `your-nuget-remote` |
| URL | `https://pkgs.dev.azure.com/{ORG}/{PROJECT}/_packaging/{FEED}/nuget/v3/index.json` |
| Username | `YOUR_ADO_USERNAME` |
| Password | `YOUR_ADO_PAT` |

Click **Test** to verify connection, then **Save**.

### How the Pipeline Works

1. **List** — Fetch all packages from Azure feed API
2. **Generate** — Dynamically create a `.csproj` with all packages as `PackageReference`
3. **Configure** — Run `jf dotnet-config` to set JFrog remote repo as resolver
4. **Restore** — Run `jf dotnet restore` — pulls packages through JFrog remote (auto-cached)
5. **Copy** — `jf rt cp` from remote cache → local repo

**When to use:** When you want JFrog to handle all protocol details natively.

---

## Key Technical Notes

| Topic | Detail |
|---|---|
| Download URL format | `pkgs.dev.azure.com/{org}/{project}/_packaging/{feed}/nuget/v3/flat2/{id}/{version}/{id}.{version}.nupkg` |
| 303 redirect | Azure NuGet API redirects to blob storage — use `-L` flag with curl |
| Package ID casing | Always **lowercase** in download URLs |
| Upload path | Flat — `jf rt u "*.nupkg" "REPO/" --flat=true` |
| Checksum deduplication | JFrog skips re-uploading files with identical SHA256 checksums (belt-and-braces on top of delta tracking) |
| Conditional upload | `condition: ne(variables['SKIP_UPLOAD'], 'true')` set via `##vso[task.setvariable]` — makes the upload task show as *skipped*, not just a no-op |
| Delta comparison | `comm -23 current.txt migrated.txt` — both files must be sorted first |
| Option 2 server ID | Use `$JFROG_CLI_SERVER_ID` in `jf dotnet-config` (not the service connection name) |
| Option 2 cache repo | JFrog remote cache is automatically named `{repo-key}-cache` |
| `jf dotnet-config` + `jf dotnet restore` | Must run in the **same `JFrogCliV2@1` task** — config file is task-scoped |

---

## Recommended Approach

| Scenario | Recommended Pipeline |
|---|---|
| See what's in the feed | Pipeline 1 — List |
| Migrate selected packages | Pipeline 2 — Migrate Specific (Option 1) |
| Full one-time migration | Pipeline 3 — Migrate All (Option 1) |
| Ongoing sync as new packages arrive | Pipeline 3 — re-run on a schedule (delta-aware) |
| On-demand proxy without pre-copying | Pipeline 4 — Remote Repo (Option 2) |
| Air-gapped environment | Option 1 — packages physically copied to JFrog |

---

## Recommended Migration Steps

Follow this order for a safe migration:

1. **Run Pipeline 1** — List all packages, verify count
2. **Test on a small feed first** — use a feed with few packages before running on production
3. **Run Pipeline 2** — Test with a curated `nuget-packages-to-sync.txt`
4. **Verify in JFrog** — confirm packages appear correctly
5. **Run Pipeline 3** — full migration once verified
6. **Re-run Pipeline 3 periodically** — only new packages added to the feed since the last run will be migrated

---

## Important Notes

- All pipelines are **read-only on Azure Artifacts** — no packages are modified or deleted
- PAT only needs **Packaging Read** scope — cannot accidentally write or delete
- Pipelines 2 and 3 write only to `migrated-packages.txt` in your own repo — no other files are touched
- Running Pipeline 2 and Pipeline 3 interchangeably is safe — they share the same tracking file

---

## Tested Environment

| Component | Version |
|---|---|
| JFrog CLI | 2.117.0 |
| Azure DevOps Agent | Mac ARM64 |
| .NET SDK | 8.0 |
| JFrog Artifactory | Cloud |