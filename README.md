# Azure Artifacts → JFrog Artifactory Migration (NuGet)

A collection of Azure DevOps pipelines to migrate **NuGet** packages from Azure Artifacts to JFrog Artifactory Cloud.

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

---

## Repository Structure

```
azure-to-jfrog-nuget-migration/
  README.md
  nuget-packages-to-sync.txt     # Curated NuGet package list for Pipeline 2
  pipelines/
    1-list-packages.yml          # List all packages in Azure NuGet feed
    2-MigrateSpecificPackages.yml       # Option 1: Migrate specific packages from list
    3-MigrateAllPackagesInFeed.yaml            # Option 1: Migrate all packages in feed
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

## Pipeline 2 — Migrate Specific Packages (Option 1)

Reads `nuget-packages-to-sync.txt` from the repo root and migrates only those listed packages to JFrog.

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
1. Read package list from `nuget-packages-to-sync.txt`
2. For each package — download `.nupkg` from Azure feed (follows 303 redirect)
3. Upload to JFrog local repository

---

## Pipeline 3 — Migrate All Packages (Option 1)

Fetches and migrates **every** package in the Azure NuGet feed. Uses pagination (100 per page) to handle large feeds.

**When to use:** For a full one-time migration or bulk sync.

**Steps:**
1. Paginate through all packages in Azure feed
2. For each package and version — download `.nupkg` (follows 303 redirect)
3. Upload all to JFrog local repository

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
| Checksum deduplication | JFrog skips re-uploading files with identical SHA256 checksums |
| Option 2 server ID | Use `$JFROG_CLI_SERVER_ID` in `jf dotnet-config` (not the service connection name) |
| Option 2 cache repo | JFrog remote cache is automatically named `{repo-key}-cache` |
| `jf dotnet-config` + `jf dotnet restore` | Must run in the **same `JFrogCliV2@1` task** — config file is task-scoped |



## Recommended Migration Steps

Follow this order for a safe migration:

1. **Run Pipeline 1** — List all packages, verify count
2. **Test on a small feed first** — use a feed with few packages before running on production
3. **Run Pipeline 2** — Test with a curated `nuget-packages-to-sync.txt`
4. **Verify in JFrog** — confirm packages appear correctly
5. **Run Pipeline 3 or 4** — full migration once verified

---

## Important Notes

- All pipelines are **read-only on Azure Artifacts** — no packages are modified or deleted
- PAT only needs **Packaging Read** scope — cannot accidentally write or delete

---

## Tested Environment

| Component | Version |
|---|---|
| JFrog CLI | 2.117.0 |
| Azure DevOps Agent | Mac ARM64 |
| .NET SDK | 8.0 |
| JFrog Artifactory | Cloud |
