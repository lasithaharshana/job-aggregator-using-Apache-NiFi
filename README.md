# Job Aggregator & Skill Matcher Pipeline
## Complete Build Guide — Apache NiFi 2.9.0

> A real-time data flow pipeline that aggregates job postings from multiple sources, normalizes their schemas, extracts skills using NLP-style keyword matching, classifies jobs by category, and routes them to specialized indexing pipelines.
Youtube Video Link:- https://youtu.be/9_RU2lHDuso
---

## 📑 Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Patterns Demonstrated](#patterns-demonstrated)
4. [Prerequisites](#prerequisites)
5. [Step-by-Step Build](#step-by-step-build)
6. [Verification](#verification)
7. [Troubleshooting](#troubleshooting)
8. [Demo Script Highlights](#demo-script-highlights)
9. [Lessons Learned](#lessons-learned)

---

## Overview

### Problem Statement

Recruitment platforms aggregate millions of job postings from hundreds of sources daily. Each source has a different schema, different freshness, and different quality. The platform must:

- Normalize disparate schemas into a canonical format
- Extract skills from unstructured job descriptions
- Classify postings by category (Software / Data / DevOps / Other)
- Route classified jobs to specialized downstream indexers
- Maintain a master archive for compliance and analytics

This is fundamentally a **data flow problem** — perfect for Apache NiFi.

### What This Pipeline Does

- **Ingests** synthetic job postings from 4 simulated sources (Indeed, LinkedIn, Remotive, ZipRecruiter)
- **Aggregates** them via a Funnel (Aggregator pattern)
- **Validates** their schema using JSONPath
- **Normalizes** them into a canonical schema
- **Enriches** them by extracting 13 skill keywords + computing seniority
- **Scores** them across 3 categories based on detected skills
- **Classifies** them into a final category
- **Routes** them to one of 4 category-specific output directories
- **Archives** every job in parallel via the Wire Tap pattern

---

## Architecture

```
┌────────────────────────────────────────────────────┐
│            Job_Aggregator_Pipeline_v1              │
│                                                    │
│  ┌─────────────────┐                               │
│  │  01_Ingestion   │   4 sources → Funnel          │
│  │                 │   → Validate → output port    │
│  └────────┬────────┘                               │
│           ↓                                        │
│  ┌─────────────────┐                               │
│  │ 02_Normalization│   Add canonical fields,       │
│  │                 │   lowercase title/desc        │
│  └────────┬────────┘                               │
│           ↓                                        │
│  ┌─────────────────┐                               │
│  │  03_Enrichment  │   Extract 13 skills,          │
│  │                 │   compute seniority,          │
│  │                 │   score 3 categories          │
│  └────────┬────────┘                               │
│           ↓                                        │
│  ┌─────────────────┐                               │
│  │04_Classification│   Classify category,          │
│  │                 │   Wire Tap → archive,         │
│  │                 │   Route 4 ways                │
│  └────────┬────────┘                               │
│           ↓                                        │
│  ┌─────────────────┐                               │
│  │   05_Output     │   4 PutFile sinks             │
│  └─────────────────┘                               │
└────────────────────────────────────────────────────┘
```

### Output Directories

| Directory | Purpose |
|-----------|---------|
| `/tmp/jobs/software/` | Software Engineering jobs |
| `/tmp/jobs/data_ml/` | Data & ML jobs |
| `/tmp/jobs/devops/` | DevOps & Cloud jobs |
| `/tmp/jobs/other/` | Uncategorized jobs |
| `/tmp/jobs/archive/` | Wire Tap — every job |
| `/tmp/jobs/errors/` | Dead Letter Queue — invalid jobs |

---

## Patterns Demonstrated

| # | Pattern | Where in Pipeline |
|---|---------|-------------------|
| 1 | **Pipes & Filters** | Every processor in the chain |
| 2 | **Aggregator** | Funnel merges 4 source streams into one |
| 3 | **Anti-Corruption Layer** | `Validate_Job_Schema` rejects malformed input |
| 4 | **Content Enricher** | `Extract_Skills`, `Compute_Seniority`, `Compute_Category_Signals` |
| 5 | **Content-Based Router** | `Route_By_Category` dispatches by job category |
| 6 | **Wire Tap** | `Archive_All_Jobs` captures every job in parallel |
| 7 | **Dead Letter Queue** | Errors directory for invalid jobs |
| 8 | **Configuration over Code** | All thresholds/keywords in Parameter Context |

---

## Prerequisites

- Docker installed and running
- Apache NiFi 2.9.0 in Docker (`apache/nifi:latest`)
- Terminal access to the NiFi container
- Browser to access NiFi UI at `https://localhost:8443/nifi`

### Start NiFi (if not running)

```bash
docker run --name nifi \
  -p 8443:8443 \
  -e SINGLE_USER_CREDENTIALS_USERNAME=admin \
  -e SINGLE_USER_CREDENTIALS_PASSWORD=adminpassword123 \
  -d apache/nifi:latest
```

Wait ~60 seconds for NiFi to start, then access `https://localhost:8443/nifi`.

### Create Output Directories

```bash
docker exec -it nifi sh -c "mkdir -p \
  /tmp/jobs/software \
  /tmp/jobs/data_ml \
  /tmp/jobs/devops \
  /tmp/jobs/other \
  /tmp/jobs/archive \
  /tmp/jobs/errors && \
  ls /tmp/jobs/"
```

Should output: `archive  data_ml  devops  errors  other  software`

---

## Step-by-Step Build

### Step 1: Create Parameter Context

**Hamburger menu (☰) → Parameter Contexts → + (plus button)**

- **Name:** `Job_Pipeline_Params`
- Click **Apply**, then **edit** the new context

Add these parameters one by one (click **+** for each):

| Name | Value |
|------|-------|
| `dir.software` | `/tmp/jobs/software` |
| `dir.data_ml` | `/tmp/jobs/data_ml` |
| `dir.devops` | `/tmp/jobs/devops` |
| `dir.other` | `/tmp/jobs/other` |
| `dir.archive` | `/tmp/jobs/archive` |
| `dir.errors` | `/tmp/jobs/errors` |
| `schedule.interval` | `5 sec` |

Click **Apply**.

---

### Step 2: Create Parent Process Group

On the main NiFi canvas:

1. Drag the **Process Group** icon (folder shape) from the toolbar
2. Name: `Job_Aggregator_Pipeline_v1`
3. Click **Add**
4. Right-click the new PG → **Configure** → **General** tab
5. Set **Process Group Parameter Context** = `Job_Pipeline_Params`
6. **Apply**
7. **Double-click** to enter

---

### Step 3: Create the 5 Child Process Groups

Inside `Job_Aggregator_Pipeline_v1`, drag the **Process Group** icon **5 times** to create:

1. `01_Ingestion`
2. `02_Normalization`
3. `03_Enrichment`
4. `04_Classification`
5. `05_Output`

Arrange left-to-right or top-to-bottom for clean visual flow.

---

### Step 4: Build `01_Ingestion` ⭐ THE FIX

**Double-click into `01_Ingestion`**

> ⚠️ **CRITICAL:** Use 4 separate generators (one per simulated source). Do NOT try to randomize a single generator's content using nested `${random()}` expressions — NiFi's Custom Text parser does not handle deeply nested expressions reliably.

#### 4.1 Create 4 Source Generators

Drag **GenerateFlowFile** processor onto canvas **4 times**.

For each, configure:
- **Settings tab:** set Name (see below)
- **Scheduling tab:** Run Schedule = `5 sec`
- **Properties tab:** Custom Text = (paste the JSON below)
- **Apply**

##### Generator 1: `Source_Indeed_Software`

```json
{"id": "${UUID()}", "title": "Senior Backend Engineer", "company": "TechCorp", "location": "Remote", "description_raw": "We need engineer with python java javascript react nodejs backend experience", "posted_date": "2026-04-26", "source": "indeed"}
```

##### Generator 2: `Source_LinkedIn_Data`

```json
{"id": "${UUID()}", "title": "Data Scientist", "company": "DataCo", "location": "Remote", "description_raw": "Looking for someone with tensorflow pandas numpy machine learning sql experience", "posted_date": "2026-04-26", "source": "linkedin"}
```

##### Generator 3: `Source_Remotive_DevOps`

```json
{"id": "${UUID()}", "title": "DevOps Engineer", "company": "CloudInc", "location": "On-site", "description_raw": "Experience with kubernetes terraform docker aws azure ci/cd required", "posted_date": "2026-04-26", "source": "remotive"}
```

##### Generator 4: `Source_ZipRecruiter_Other`

```json
{"id": "${UUID()}", "title": "Marketing Manager", "company": "BrandX", "location": "On-site", "description_raw": "Marketing manager with social media branding content strategy experience", "posted_date": "2026-04-26", "source": "ziprecruiter"}
```

> 💡 **Why 4 generators?** This implements the **Aggregator pattern** — each generator simulates a different real-world job source. Adding a 5th source in production is just adding a 5th GenerateFlowFile (or replacing one with `InvokeHTTP` calling a real API).

#### 4.2 Add a Funnel (Aggregator)

From the top toolbar, drag the **Funnel** icon (small inverted-triangle/cone) onto the canvas. This is NiFi's native way to merge multiple streams.

#### 4.3 Add `Validate_Job_Schema` (EvaluateJsonPath)

Drag a Processor → search **`EvaluateJsonPath`** → Add.

- **Settings tab:** Name = `Validate_Job_Schema`
- **Properties tab:**
  - **Destination:** `flowfile-attribute`
  - **Return Type:** `auto-detect`
  - Click **+** to add custom properties:

| Property Name | Value |
|---------------|-------|
| `job.id` | `$.id` |
| `job.title` | `$.title` |
| `job.company` | `$.company` |
| `job.location` | `$.location` |
| `job.description` | `$.description_raw` |
| `job.source` | `$.source` |

- **Relationships tab:** ✅ check `terminate` for `unmatched`
- **Apply**

#### 4.4 Add `Log_Invalid_Jobs` (PutFile) — Dead Letter Queue

Drag a **PutFile** processor onto canvas.

- Name: `Log_Invalid_Jobs`
- **Properties:**
  - Directory: `#{dir.errors}`
  - Conflict Resolution Strategy: `replace`
- **Relationships tab:** ✅ check `terminate` for both `success` and `failure`

#### 4.5 Add Output Port

Drag the **Output Port** icon → Name: `jobs_validated_out` → Add.

#### 4.6 Connect Everything

- `Source_Indeed_Software` → Funnel (`success`)
- `Source_LinkedIn_Data` → Funnel (`success`)
- `Source_Remotive_DevOps` → Funnel (`success`)
- `Source_ZipRecruiter_Other` → Funnel (`success`)
- Funnel → `Validate_Job_Schema` (no dialog appears)
- `Validate_Job_Schema` → `jobs_validated_out` (`matched`)
- `Validate_Job_Schema` → `Log_Invalid_Jobs` (`failure` AND `unmatched` — check both boxes)

#### 4.7 Visual Layout

```
Source_Indeed_Software ────┐
Source_LinkedIn_Data ──────┤
                           ├──> Funnel ──> Validate_Job_Schema ──> jobs_validated_out
Source_Remotive_DevOps ────┤                       │
Source_ZipRecruiter_Other ─┘                       └──> Log_Invalid_Jobs
```

✅ **CHECKPOINT:** `01_Ingestion` complete. Click breadcrumb to go back.

---

### Step 5: Build `02_Normalization`

**Double-click into `02_Normalization`**

#### 5.1 Add Input Port

Drag Input Port → Name: `jobs_in` → Add.

#### 5.2 Add `Normalize_Schema` (UpdateAttribute)

Drag Processor → search `UpdateAttribute` → Add.

- **Settings tab:** Name = `Normalize_Schema`
- **Properties tab:** Click **+** for each:

| Property Name | Value |
|---------------|-------|
| `job.title.lower` | `${job.title:toLower()}` |
| `job.description.lower` | `${job.description:toLower()}` |
| `job.is.remote` | `${job.location:equalsIgnoreCase('Remote')}` |
| `job.normalized.timestamp` | `${now():format("yyyy-MM-dd'T'HH:mm:ss'Z'")}` |

#### 5.3 Add Output Port

Drag Output Port → Name: `jobs_normalized_out` → Add.

#### 5.4 Connect

- `jobs_in` → `Normalize_Schema` (`success`)
- `Normalize_Schema` → `jobs_normalized_out` (`success`)

✅ Go back to parent.

---

### Step 6: Build `03_Enrichment` ⭐ THE NLP STAR

**Double-click into `03_Enrichment`**

#### 6.1 Add Input Port

Drag Input Port → Name: `jobs_in` → Add.

#### 6.2 Add `Extract_Skills` (UpdateAttribute)

- **Settings tab:** Name = `Extract_Skills`
- **Properties tab:** Add 13 boolean skill flags:

| Property Name | Value |
|---------------|-------|
| `has.python` | `${job.description.lower:contains('python')}` |
| `has.java` | `${job.description.lower:contains('java')}` |
| `has.javascript` | `${job.description.lower:contains('javascript')}` |
| `has.react` | `${job.description.lower:contains('react')}` |
| `has.nodejs` | `${job.description.lower:contains('nodejs')}` |
| `has.aws` | `${job.description.lower:contains('aws')}` |
| `has.docker` | `${job.description.lower:contains('docker')}` |
| `has.kubernetes` | `${job.description.lower:contains('kubernetes')}` |
| `has.terraform` | `${job.description.lower:contains('terraform')}` |
| `has.tensorflow` | `${job.description.lower:contains('tensorflow')}` |
| `has.pandas` | `${job.description.lower:contains('pandas')}` |
| `has.ml` | `${job.description.lower:contains('machine learning')}` |
| `has.cicd` | `${job.description.lower:contains('ci/cd')}` |

#### 6.3 Add `Compute_Seniority` (UpdateAttribute)

- **Settings tab:** Name = `Compute_Seniority`
- **Properties tab:**

| Property Name | Value |
|---------------|-------|
| `is.senior.title` | `${job.title.lower:contains('senior')}` |
| `is.lead.title` | `${job.title.lower:contains('lead')}` |
| `is.junior.title` | `${job.title.lower:contains('junior')}` |
| `seniority.level` | `${job.title.lower:contains('senior'):equals('true'):ifElse('SENIOR',${job.title.lower:contains('lead'):equals('true'):ifElse('LEAD',${job.title.lower:contains('junior'):equals('true'):ifElse('JUNIOR','MID')})})}` |

#### 6.4 Add `Compute_Category_Signals` (UpdateAttribute)

- **Settings tab:** Name = `Compute_Category_Signals`
- **Properties tab:** Score each category:

| Property Name | Value |
|---------------|-------|
| `score.software` | `${literal(0):plus(${has.python:equals('true'):ifElse(1,0)}):plus(${has.java:equals('true'):ifElse(1,0)}):plus(${has.javascript:equals('true'):ifElse(1,0)}):plus(${has.react:equals('true'):ifElse(1,0)}):plus(${has.nodejs:equals('true'):ifElse(1,0)})}` |
| `score.data_ml` | `${literal(0):plus(${has.tensorflow:equals('true'):ifElse(1,0)}):plus(${has.pandas:equals('true'):ifElse(1,0)}):plus(${has.ml:equals('true'):ifElse(1,0)})}` |
| `score.devops` | `${literal(0):plus(${has.aws:equals('true'):ifElse(1,0)}):plus(${has.docker:equals('true'):ifElse(1,0)}):plus(${has.kubernetes:equals('true'):ifElse(1,0)}):plus(${has.terraform:equals('true'):ifElse(1,0)}):plus(${has.cicd:equals('true'):ifElse(1,0)})}` |

#### 6.5 Add Output Port

Drag Output Port → Name: `jobs_enriched_out` → Add.

#### 6.6 Connect

```
jobs_in → Extract_Skills → Compute_Seniority → Compute_Category_Signals → jobs_enriched_out
```

(All connections on `success`.)

✅ Go back to parent.

---

### Step 7: Build `04_Classification` ⭐ ROUTER + WIRE TAP

**Double-click into `04_Classification`**

#### 7.1 Add Input Port

Drag Input Port → Name: `jobs_in` → Add.

#### 7.2 Add `Classify_Category` (UpdateAttribute)

- **Settings tab:** Name = `Classify_Category`
- **Properties tab:**

| Property Name | Value |
|---------------|-------|
| `job.category` | `${score.devops:toNumber():gt(${score.software:toNumber()}):and(${score.devops:toNumber():gt(${score.data_ml:toNumber()})}):equals('true'):ifElse('DEVOPS_CLOUD',${score.data_ml:toNumber():gt(${score.software:toNumber()}):equals('true'):ifElse('DATA_ML',${score.software:toNumber():gt(0):equals('true'):ifElse('SOFTWARE_ENGINEERING','OTHER')})})}` |

> 💡 **Logic:** Highest score wins. Tie-breaker order: DevOps > Data > Software > Other.

#### 7.3 Add `Route_By_Category` (RouteOnAttribute)

Drag Processor → search `RouteOnAttribute` → Add.

- **Settings tab:** Name = `Route_By_Category`
- **Properties tab:**
  - **Routing Strategy:** `Route to Property name`
  - Click **+** for each route:

| Property Name | Value |
|---------------|-------|
| `software` | `${job.category:equals('SOFTWARE_ENGINEERING')}` |
| `data_ml` | `${job.category:equals('DATA_ML')}` |
| `devops` | `${job.category:equals('DEVOPS_CLOUD')}` |
| `other` | `${job.category:equals('OTHER')}` |

- **Relationships tab:** ✅ check `terminate` for `unmatched`

#### 7.4 Add `Archive_All_Jobs` (PutFile) — WIRE TAP

- Name: `Archive_All_Jobs`
- **Properties:**
  - Directory: `#{dir.archive}`
  - Conflict Resolution Strategy: `replace`
- **Relationships tab:** ✅ check `terminate` for both `success` and `failure`

#### 7.5 Add 4 Output Ports

- `jobs_software_out`
- `jobs_data_ml_out`
- `jobs_devops_out`
- `jobs_other_out`

#### 7.6 Connect — THE WIRE TAP FORK

```
jobs_in → Classify_Category
              │
              ├──> Route_By_Category ──> 4 output ports (by category)
              │
              └──> Archive_All_Jobs (Wire Tap)
```

> ⚠️ **CRITICAL:** From `Classify_Category`, draw **TWO** outgoing connections — one to `Route_By_Category` and one to `Archive_All_Jobs`. Both use the `success` relationship. This fork is what creates the Wire Tap pattern.

Then from router:
- `Route_By_Category` → `jobs_software_out` (relationship: `software`)
- `Route_By_Category` → `jobs_data_ml_out` (relationship: `data_ml`)
- `Route_By_Category` → `jobs_devops_out` (relationship: `devops`)
- `Route_By_Category` → `jobs_other_out` (relationship: `other`)

✅ Go back to parent.

---

### Step 8: Build `05_Output`

**Double-click into `05_Output`**

#### 8.1 Add 4 Input Ports

- `jobs_software_in`
- `jobs_data_ml_in`
- `jobs_devops_in`
- `jobs_other_in`

#### 8.2 Add 4 PutFile Sinks

For each: drag PutFile, configure, add.

| Processor Name | Directory | Terminate |
|----------------|-----------|-----------|
| `Save_Software_Jobs` | `#{dir.software}` | success + failure |
| `Save_Data_ML_Jobs` | `#{dir.data_ml}` | success + failure |
| `Save_DevOps_Jobs` | `#{dir.devops}` | success + failure |
| `Save_Other_Jobs` | `#{dir.other}` | success + failure |

For all: Conflict Resolution Strategy = `replace`.

#### 8.3 Connect

- `jobs_software_in` → `Save_Software_Jobs`
- `jobs_data_ml_in` → `Save_Data_ML_Jobs`
- `jobs_devops_in` → `Save_DevOps_Jobs`
- `jobs_other_in` → `Save_Other_Jobs`

✅ Go back to parent.

---

### Step 9: Connect Process Groups

You're now on the parent canvas with all 5 PGs visible.

Hover over the **right edge** of `01_Ingestion` → green arrow appears → drag to `02_Normalization` → dialog asks to pick output port (`jobs_validated_out`) and input port (`jobs_in`) → Add.

Repeat for:

| From | Output Port | To | Input Port |
|------|-------------|----|------------|
| `01_Ingestion` | `jobs_validated_out` | `02_Normalization` | `jobs_in` |
| `02_Normalization` | `jobs_normalized_out` | `03_Enrichment` | `jobs_in` |
| `03_Enrichment` | `jobs_enriched_out` | `04_Classification` | `jobs_in` |
| `04_Classification` | `jobs_software_out` | `05_Output` | `jobs_software_in` |
| `04_Classification` | `jobs_data_ml_out` | `05_Output` | `jobs_data_ml_in` |
| `04_Classification` | `jobs_devops_out` | `05_Output` | `jobs_devops_in` |
| `04_Classification` | `jobs_other_out` | `05_Output` | `jobs_other_in` |

✅ All 5 process groups linked. No dangling ports. No yellow warnings.

---

### Step 10: Pre-flight Check

Before starting:

1. ✅ No yellow warning triangles anywhere
2. ✅ All processors stopped (red squares)
3. ✅ All queues empty
4. ✅ Parameter Context bound (right-click parent PG → Configure → General → confirm `Job_Pipeline_Params`)

If any yellow triangle appears:
- Click it — it tells you what's missing
- Most common cause: an unterminated relationship — right-click processor → Configure → Relationships → check `terminate`

---

## Verification

### Start the Pipeline

Right-click the parent canvas → **Start**. All processors turn green.

### Wait 60 seconds, then run

```bash
echo "Software: $(docker exec -i nifi sh -c 'ls /tmp/jobs/software | wc -l')"
echo "Data/ML:  $(docker exec -i nifi sh -c 'ls /tmp/jobs/data_ml | wc -l')"
echo "DevOps:   $(docker exec -i nifi sh -c 'ls /tmp/jobs/devops | wc -l')"
echo "Other:    $(docker exec -i nifi sh -c 'ls /tmp/jobs/other | wc -l')"
echo "Archive:  $(docker exec -i nifi sh -c 'ls /tmp/jobs/archive | wc -l')"
echo "Errors:   $(docker exec -i nifi sh -c 'ls /tmp/jobs/errors | wc -l')"
```

### Expected Output

```
Software: ~12
Data/ML:  ~12
DevOps:   ~12
Other:    ~12
Archive:  ~48
Errors:   0
```

A **balanced 25%/25%/25%/25%** distribution — each source fires every 5 seconds → ~12 jobs/minute → routed to its category.

### Read Sample Files

```bash
# Software example
docker exec -it nifi sh -c "ls /tmp/jobs/software | head -1 | xargs -I {} cat /tmp/jobs/software/{}"

# Data/ML example
docker exec -it nifi sh -c "ls /tmp/jobs/data_ml | head -1 | xargs -I {} cat /tmp/jobs/data_ml/{}"

# DevOps example
docker exec -it nifi sh -c "ls /tmp/jobs/devops | head -1 | xargs -I {} cat /tmp/jobs/devops/{}"

# Other example
docker exec -it nifi sh -c "ls /tmp/jobs/other | head -1 | xargs -I {} cat /tmp/jobs/other/{}"
```

You should see clean JSON with real values (not literal `${random()...}` text).

### Verify Wire Tap

```bash
SUM=$(($(docker exec -i nifi sh -c 'ls /tmp/jobs/software | wc -l') + \
       $(docker exec -i nifi sh -c 'ls /tmp/jobs/data_ml | wc -l') + \
       $(docker exec -i nifi sh -c 'ls /tmp/jobs/devops | wc -l') + \
       $(docker exec -i nifi sh -c 'ls /tmp/jobs/other | wc -l')))
ARCHIVE=$(docker exec -i nifi sh -c 'ls /tmp/jobs/archive | wc -l')
echo "Sum of categories: $SUM"
echo "Archive (Wire Tap): $ARCHIVE"
```

`Sum` and `Archive` should be approximately equal (small difference is normal — the two branches run independently).

---

## Troubleshooting

### Issue: All jobs go to ONE category (e.g., DevOps)

**Cause:** GenerateFlowFile's Custom Text contains nested `${random()}` expressions which NiFi does not evaluate reliably. The literal expression text appears in the description, and ALL keywords match.

**Fix:** Use 4 separate generators (one per source) with static, simple JSON. This is what Step 4 above teaches.

**How to confirm:** Read an archive file. If you see `${random():mod(8)...}` text instead of real values, this is the bug.

### Issue: Errors directory has files

**Diagnose:**
```bash
docker exec -it nifi sh -c "ls /tmp/jobs/errors | head -1 | xargs -I {} cat /tmp/jobs/errors/{}"
```

Most likely cause: malformed JSON in a generator's Custom Text. Check that all braces match and there are no unescaped quotes.

### Issue: Yellow warning triangle on a processor

Click the triangle — NiFi tells you exactly what's missing. Most common causes:
- Unterminated relationship → Configure → Relationships → check `terminate`
- Missing required property → Configure → Properties → fill in
- Parameter Context not bound → Right-click parent PG → Configure → General → set Parameter Context

### Issue: Queues filling up but no output files

Inspect the queue: right-click queue → **List queue**. Click a flowfile to see attributes. Check downstream processor — most likely it's stopped or has a property error.

### Issue: NiFi container won't start / port conflict

```bash
# Stop existing container
docker stop nifi && docker rm nifi

# Start fresh
docker run --name nifi -p 8443:8443 \
  -e SINGLE_USER_CREDENTIALS_USERNAME=admin \
  -e SINGLE_USER_CREDENTIALS_PASSWORD=adminpassword123 \
  -d apache/nifi:latest

# Wait 60 seconds for startup, then recreate output dirs
sleep 60
docker exec -it nifi sh -c "mkdir -p /tmp/jobs/{software,data_ml,devops,other,archive,errors}"
```

> ⚠️ Note: starting a fresh container loses all your flow definitions. Only do this if necessary. To persist flows across container restarts, use volume mounts:
>
> ```bash
> docker run --name nifi -p 8443:8443 \
>   -v nifi_flow:/opt/nifi/nifi-current/conf \
>   -e SINGLE_USER_CREDENTIALS_USERNAME=admin \
>   -e SINGLE_USER_CREDENTIALS_PASSWORD=adminpassword123 \
>   -d apache/nifi:latest
> ```

### Reset for Fresh Demo

```bash
docker exec -it nifi sh -c "rm -rf \
  /tmp/jobs/software/* \
  /tmp/jobs/data_ml/* \
  /tmp/jobs/devops/* \
  /tmp/jobs/other/* \
  /tmp/jobs/archive/* \
  /tmp/jobs/errors/*"
```

---

## Demo Script Highlights

### Opening (30 sec)

> "Recruitment platforms aggregate millions of job postings from hundreds of sources daily — Indeed, LinkedIn, Remotive, ZipRecruiter — each with a different schema. The platform must normalize, classify, and route them in real time. This is fundamentally a data flow problem, which is why I built it using Apache NiFi."

### Architecture (45 sec)

> "My pipeline implements eight enterprise integration patterns. The Aggregator pattern — a Funnel — merges four source streams into one. The Anti-Corruption Layer rejects malformed input at the boundary. The Content Enricher extracts thirteen skill keywords and classifies seniority. The Content-Based Router dispatches by category. And the Wire Tap forks every classified job into a master archive in parallel with the routing path."

### The Funnel Talking Point (gold-standard)

> "Adding a fifth job source in production is just adding a fifth GenerateFlowFile processor — or replacing one with InvokeHTTP calling a real API like Remotive's public endpoint. Zero changes to anything downstream. That's the value of the Aggregator pattern."

### Live Verification Moment

Run the verification command on screen. Show the balanced 25%/25%/25%/25% distribution. Open one job from each category to prove the routing is correct.

### Closing (15 sec)

> "In production, I would replace the synthetic generators with InvokeHTTP processors calling real job APIs, swap the keyword-matching skill extraction for a hosted ML model, and replace the file sinks with Kafka producers feeding downstream search indexers. The architecture supports each substitution without touching the rest of the flow — which is precisely the value of choosing data flow architecture for this problem."

---

## Lessons Learned

### What Went Wrong First Time

**The bug:** A single GenerateFlowFile with nested `${random():mod(8):toString():equals('0'):ifElse(...)}` expressions in Custom Text. NiFi only evaluated `${UUID()}` correctly; the nested expressions were left as literal text.

**The symptom:** Every job had identical literal text in its description. Since that text contained ALL keywords (python, kubernetes, terraform, etc.), every score was equal, and DevOps won the tie. **All 115 jobs → DevOps.** Software, Data/ML, and Other were all zero.

**The lesson:** NiFi Custom Text supports simple EL functions but does not handle deeply nested `:ifElse(${...})` chains reliably. **Use multiple generators with static content** instead of trying to randomize with one.

### Key Takeaways

1. **Test EL expressions in isolation** before embedding in Custom Text. Use a single UpdateAttribute first to verify the expression evaluates correctly.

2. **Read the actual file output** as soon as anything generates. If you see literal `${...}` text in your output, the EL didn't evaluate.

3. **Multiple generators ≠ less elegant.** In this case, 4 separate generators map directly to "4 different job sources" 

4. **Funnels are underrated.** They are NiFi's native Aggregator pattern implementation and create stunning visual flow diagrams.

5. **Always have a Wire Tap and a Dead Letter Queue.** These are free patterns 

### Why This Pipeline Is Production-Ready (with substitutions)

| Demo Component | Production Replacement |
|----------------|------------------------|
| GenerateFlowFile sources | `InvokeHTTP` against real APIs |
| Keyword skill matching | Hosted ML model via `InvokeHTTP` |
| `PutFile` sinks | `PublishKafkaRecord_2_6` to downstream microservices |
| Local Docker | Clustered NiFi behind load balancer |
| Hardcoded categories | ML-based dynamic categorization |
| Static skills list | Skills database with periodic refresh |

The architecture itself does not change — only the I/O processors do. **This is the value of data flow architecture.**

---

## Quick Reference Card

### Verification one-liner

```bash
echo "Software: $(docker exec -i nifi sh -c 'ls /tmp/jobs/software | wc -l')"; \
echo "Data/ML:  $(docker exec -i nifi sh -c 'ls /tmp/jobs/data_ml | wc -l')"; \
echo "DevOps:   $(docker exec -i nifi sh -c 'ls /tmp/jobs/devops | wc -l')"; \
echo "Other:    $(docker exec -i nifi sh -c 'ls /tmp/jobs/other | wc -l')"; \
echo "Archive:  $(docker exec -i nifi sh -c 'ls /tmp/jobs/archive | wc -l')"; \
echo "Errors:   $(docker exec -i nifi sh -c 'ls /tmp/jobs/errors | wc -l')"
```

### Reset one-liner

```bash
docker exec -it nifi sh -c "rm -rf /tmp/jobs/{software,data_ml,devops,other,archive,errors}/*"
```

### Read sample one-liner

```bash
docker exec -it nifi sh -c "ls /tmp/jobs/software | head -1 | xargs -I {} cat /tmp/jobs/software/{}"
```

---
