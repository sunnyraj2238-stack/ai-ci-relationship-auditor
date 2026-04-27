[README.md](https://github.com/user-attachments/files/27119680/README.md)
# AI CI Relationship Auditor

> **Automated CMDB relationship discovery powered by Groq LLaMA 3.3 70B — built on ServiceNow.**

[![ServiceNow](https://img.shields.io/badge/ServiceNow-Australia%20Release-green?logo=servicenow&logoColor=white)](https://www.servicenow.com)
[![Groq](https://img.shields.io/badge/LLM-Groq%20LLaMA%203.3%2070B-orange)](https://groq.com)
[![Scope](https://img.shields.io/badge/App%20Scope-x__1412526__ai__ci__re-blue)](#)
[![Challenge](https://img.shields.io/badge/%23BuildMoreWithBuildAgent-ServiceNow%20Community-purple)](https://www.servicenow.com/community/developer-advocate-blog/more-calls-more-builds-the-buildmorewithbuildagent-challenge-is/ba-p/3525083)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](#license)

---

## The Problem

ServiceNow Discovery populates Configuration Items (CIs) reliably. **Relationships between those CIs? Not so much.**

Most enterprise CMDB environments accumulate *relationship debt* — servers, databases, and application components that exist in the CMDB but whose connections to one another are incomplete, stale, or simply missing. The impact is real:

- Service maps with gaps that make impact analysis unreliable
- Incomplete upstream/downstream dependency chains during incident response
- CSDM 4.0 compliance scores that stay amber despite clean CI populations
- Manual topology documentation that drifts from reality within weeks

The **AI CI Relationship Auditor** solves this by turning raw discovery log output into validated, CMDB-ready relationships automatically — using a large language model to do the pattern recognition that previously required a human CMDB architect.

---

## Solution Overview

A scoped ServiceNow application with a **three-stage AI pipeline**:

```
Discovery Log  ──►  LLM Extraction  ──►  IRE Validation  ──►  CMDB Write
(Log Staging)       (Groq LLaMA 3.3)    (CI resolve +        (cmdb_rel_ci)
                                          dedup check)
```

Each candidate relationship is scored with a **confidence value (0.0–1.0)**. High-confidence results are written to `cmdb_rel_ci` automatically; medium-confidence ones are queued for human review; low-confidence and invalid results are logged and rejected.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AI CI Relationship Auditor                       │
│                        Scope: x_1412526_ai_ci_re                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────┐  │
│   │  Log Staging │    │  CMDB CI List    │    │  Groq API           │  │
│   │  (table)     │───►│  200 CIs α-sort  │───►│  LLaMA 3.3 70B      │  │
│   └──────────────┘    └──────────────────┘    └─────────────────────┘  │
│                                                          │              │
│                                              JSON candidate array       │
│                                                          ▼              │
│                                        ┌─────────────────────────────┐ │
│  Stage 1                               │     CIAudLLMParser          │ │
│                                        │  Extract + score candidates  │ │
│                                        └──────────────┬──────────────┘ │
│                                                       │                │
│                                        ┌──────────────▼──────────────┐ │
│  Stage 2                               │     CIAudIREValidator        │ │
│                                        │  Resolve CIs · Dedup check   │ │
│                                        └──────────────┬──────────────┘ │
│                                                       │                │
│                                        ┌──────────────▼──────────────┐ │
│  Stage 3                               │     CIAudCMDBWriter          │ │
│                                        │  Route by confidence tier    │ │
│                                        └──────┬──────────────┬────────┘ │
│                                               │              │          │
│                          ┌────────────────────▼──┐    ┌──────▼───────┐ │
│                          │  cmdb_rel_ci           │    │  Candidate   │ │
│                          │  (≥ 0.85 auto-write)   │    │  Queue       │ │
│                          └───────────────────────┘    │  (0.60–0.84) │ │
│                                                        └──────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Confidence Tiers

| Tier | Range | Action |
|------|-------|--------|
| **High** | ≥ 0.85 | Auto-write to `cmdb_rel_ci` |
| **Medium** | 0.60 – 0.84 | Stage in `Relationship Candidate` table for human review |
| **Low / Invalid** | < 0.60 or fails validation | Write to `CI Audit Log` with reason |

---

## App Components

### Script Includes

| Name | Purpose |
|------|---------|
| `CIAudLLMParser` | Fetches 200 named CIs from CMDB, builds a structured prompt, calls Groq API, parses JSON response into candidate array |
| `CIAudIREValidator` | Resolves source/target CI sys_ids from names, maps relationship types to `cmdb_rel_type`, checks for existing duplicates in `cmdb_rel_ci` |
| `CIAudCMDBWriter` | Inserts validated candidates to the `Relationship Candidate` table; auto-writes High-confidence ones to `cmdb_rel_ci`; writes audit log entries |

### Custom Tables

| Table | Label | Purpose |
|-------|-------|---------|
| `x_1412526_ai_ci_re_log_staging` | Log Staging | Input queue for raw discovery log text |
| `x_1412526_ai_ci_re_relationship_candidate` | Relationship Candidate | Staging area for Medium-confidence relationships pending review |
| `x_1412526_ai_ci_re_ci_audit_log` | CI Audit Log | Immutable audit trail for every pipeline action |

### Performance Analytics

| Indicator | Metric |
|-----------|--------|
| CI Auditor — Candidates Extracted (7d) | Count of candidates staged in last 7 days |
| CI Auditor — Auto-Written Relationships | Count of records written to `cmdb_rel_ci` |
| CI Auditor — Pending Human Review | Count of Medium-confidence candidates awaiting action |
| CI Auditor — Avg Confidence Score | Average confidence score across all staged candidates |
| CI Auditor — Logs Processed (24h) | Count of Log Staging records in `Processed` status |
| CI Auditor — Rejected Candidates | Count of candidates with `Rejected` status |

All indicators use **Real Time collection** via the `AIAuditor.RelationshipCandidate` Indicator Source, surfaced in an Analytics Hub Health Dashboard.

### Supporting Artifacts

- **Business Rule** — `CI Auditor — Auto-populate Confidence Ti`: Auto-sets confidence tier label on `Relationship Candidate` insert
- **UI Actions** — `Approve Candidate` / `Reject Candidate`: One-click human review workflow on the Candidate table
- **System Properties** — 6 scoped properties (see [Configuration](#configuration))

---

## Prerequisites

- ServiceNow instance (PDI or sandbox) on **Washington DC release or later** (tested on Australia)
- **Admin** role for Update Set import
- A free **Groq API key** from [console.groq.com](https://console.groq.com) — no cost for the LLaMA 3.3 70B model tier used here
- Performance Analytics plugin activated (pre-installed on most instances)

---

## Installation

### Option 1 — Update Set XML (Recommended)

1. Download the latest `AI_CI_Relationship_Auditor_v1.0.xml` from [Releases](../../releases)
2. In your ServiceNow instance navigate to **System Update Sets → Retrieved Update Sets**
3. Click **Import Update Set from XML** and upload the file
4. Open the retrieved update set and click **Preview Update Set**
5. Resolve any conflicts (there should be none on a clean instance), then click **Commit Update Set**
6. Verify the app appears in **App Engine Studio** under scope `x_1412526_ai_ci_re`

### Option 2 — Source Control (App Engine Studio)

1. Open **App Engine Studio** on your instance
2. Click **Import from Source Control**
3. Enter repository URL: `https://github.com/sunnyraj2238-stack/ai-ci-relationship-auditor`
4. Authenticate with your GitHub credentials and import

---

## Configuration

After installation, set these system properties in **System Properties** (navigate to `sys_properties.list`):

| Property | Required | Description | Example |
|----------|----------|-------------|---------|
| `x_1412526_ai_ci_re.groq.api_key` | ✅ Yes | Your Groq API key | `gsk_...` |
| `x_1412526_ai_ci_re.groq.model` | Optional | Groq model ID | `llama-3.3-70b-versatile` |
| `x_1412526_ai_ci_re.groq.max_tokens` | Optional | Max tokens per API call | `1500` |
| `x_1412526_ai_ci_re.confidence.high` | Optional | Auto-write threshold | `0.85` |
| `x_1412526_ai_ci_re.confidence.low` | Optional | Minimum acceptance threshold | `0.60` |
| `x_1412526_ai_ci_re.review.group` | Optional | Assignment group for human review queue | `CMDB Architects` |

> **Groq API key**: Sign up at [console.groq.com](https://console.groq.com). The `llama-3.3-70b-versatile` model is free tier eligible.

---

## Usage

### 1. Stage a Discovery Log

Navigate to **AI CI Relationship Auditor → Log Staging → New** and paste raw discovery log text into the `Log Content` field. Save with status `New`.

Or create one via script:

```javascript
var gr = new GlideRecord('x_1412526_ai_ci_re_log_staging');
gr.log_content = '... your discovery log text ...';
gr.status = 'New';
gr.source = 'Manual';
gr.insert();
```

### 2. Run the Pipeline

Open a **Background Script** in scope `AI CI Relationship Auditor` and run:

```javascript
var logId = '<sys_id of your Log Staging record>';

var parser    = new CIAudLLMParser();
var validator = new CIAudIREValidator();
var writer    = new CIAudCMDBWriter();

var candidates = parser.parseLog(logId);

candidates.forEach(function(cand) {
    var result = validator.validate(cand);
    if (result.valid) {
        writer.stageCandidate(cand, result, logId);
    }
});

gs.info('Pipeline complete. Candidates processed: ' + candidates.length);
```

### 3. Review Medium-Confidence Candidates

Navigate to **AI CI Relationship Auditor → Relationship Candidates**. Use the **Approve** or **Reject** UI Actions on any `Pending` record. Approved records are written to `cmdb_rel_ci`.

### 4. Monitor via Dashboard

Open **Analytics Hub → CI Relationship Auditor — Health Dashboard** to see all 6 KPI indicators in real time.

---

## Sample Pipeline Results

Tested against a live PDI CMDB with SAP and Linux/MySQL topologies:

| Discovery Log | CIs Referenced | Extracted | Auto-Written | Pending Review | Invalid |
|--------------|---------------|-----------|-------------|---------------|---------|
| SAP Web Tier | WEB01-04, AppSRV01/02, ORA01 | 8 | 6 | 0 | 2 (already existed) |
| SAP LB & SD Cluster | LoadBal01/02, SD-01/02/03, ORA01 | 8 | 4 | 4 | 0 |
| Linux / MySQL Tier | LinuxApp01/02, LinuxHost-01, Apache×2, MySQL-16011/14/16 | 11 | 11 | 0 | 0 |
| **Total** | **13 distinct CIs** | **27** | **21** | **4** | **2** |

Service Mapping layer recomputation was triggered automatically on every `cmdb_rel_ci` insert — confirming topological validity.

---

## How Build Agent Was Used

This app was developed using **ServiceNow Build Agent** (Now IDE AI assistant) to accelerate development. Build Agent assisted with:

- Scaffolding the initial Script Include structures for all three pipeline stages
- Generating the confidence-tier routing logic in `CIAudCMDBWriter`
- Drafting the Business Rule for auto-populating the confidence tier label
- Suggesting the PA Indicator source configuration for Real Time collection

Build Agent's natural language prompting dramatically reduced time spent on ServiceNow boilerplate — allowing focus on prompt engineering, validation logic, and CMDB architecture.

---

## Relationship Types Supported

The validator maps free-text relationship descriptions from the LLM to these `cmdb_rel_type` values:

| LLM Output | ServiceNow `cmdb_rel_type` |
|------------|---------------------------|
| `Runs on` | Runs on::Runs |
| `Depends on` | Depends on::Used by |
| `Connects to` | Connects to::Connected by |
| `Uses` | Uses::Used by |
| `Hosted on` | Hosted on::Hosts |

---

## Known Limitations

- `_getKnownCIs()` caps at 200 CIs per pipeline run. For CMDBs with thousands of relevant CIs, consider pre-filtering by `sys_class_name` or `support_group` to pass the most relevant subset to the LLM.
- Groq API rate limits apply on the free tier. For high-volume runs, add retry logic or upgrade your Groq plan.
- The pipeline does not currently handle multi-hop relationships (A → B → C inferred from a single log). Each relationship is treated independently.
- Build Agent (Trial) on PDIs requires Now Assist to be activated via Admin Center.

---

## Roadmap

- [ ] Scheduled job to automatically process all `New` log staging records
- [ ] Slack / Teams notification when High-confidence relationships are auto-written
- [ ] Support for additional LLM providers (OpenAI, Gemini) via provider-agnostic wrapper
- [ ] Bulk review UI for the Relationship Candidate table
- [ ] CSDM 4.0 relationship type mapping alignment

---

## Contributing

Pull requests welcome. Please open an issue first to discuss significant changes.

1. Fork the repo
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Open a Pull Request

---

## Author

**Raju Munta** — ServiceNow Solutions Architect & ITSM Consultant  
Specialising in Enterprise ServiceNow: ITSM, ITOM, CMDB, CSDM 4.0, ITAM, CSM

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Raju%20Munta-0077B5?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/rajumunta/)
[![ServiceNow Community](https://img.shields.io/badge/ServiceNow%20Community-Profile-green?logo=servicenow&logoColor=white)](https://www.servicenow.com/community/user/viewprofilepage/user-id/319066)
[![Portfolio](https://img.shields.io/badge/Portfolio-rajumunta.me-informational?logo=googlechrome&logoColor=white)](https://rajumunta.me)

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*Built with ServiceNow Build Agent · Groq LLaMA 3.3 70B · App Engine Studio · Australia Release*
