# Architecture Deep Dive

A node-by-node breakdown of the hirescout workflow, design decisions, and notes from running this in production at Inoma Digital.

---

## System Overview

```
                  ┌──────────────────────────────────────┐
                  │         Hetzner VPS (Ubuntu)          │
                  │                                       │
 Google Sheets ──►│  Nginx ──► n8n (Docker :5678)        │
 webhook          │                 │                     │
                  │                 │──► Resume Parser    │
                  │                      (custom API)     │
                  └──────────────────────────────────────┘
                                    │
                       ┌────────────┼────────────┐
                       │            │             │
                  Google Drive  Gemini API   Notion API
```

The VPS runs two services: n8n (workflow engine) and a custom resume parser microservice. Both sit behind Nginx with SSL. Cloudflare sits in front of the domain, which is why the webhook payload includes `cf-ray`, `cf-ipcountry`, and related headers.

---

## Node-by-Node Breakdown

### 1. Webhook

Receives POST requests from Google Sheets via a Google Apps Script trigger on form submission. The payload body contains all form field responses as flat key-value pairs, plus standard HTTP headers including Cloudflare proxy headers.

---

### 2. If — Validity Gate

**Condition:** `candidate_name` is not empty

The simplest possible spam filter. Empty or malformed webhook hits (test pings, Apps Script misfires, blank submissions) are dropped here before any processing begins. Only submissions with a non-empty candidate name proceed.

This single condition is intentional — the Google Form itself enforces required fields, so by the time data reaches the webhook, a missing name is the clearest signal of a bad payload.

---

### 3. Normalize Form Data

Maps raw webhook body fields into a clean, consistently named object. Google Forms field names come through verbatim (spaces, question marks, capitalization), so this node creates a stable internal schema the rest of the workflow depends on.

---

### 4. CV Branch (parallel)

Runs alongside the AI branch after normalization.

**Wait** — brief pause to allow Google Drive to finish processing the uploaded file before attempting access.

**Process Links for Drive** — extracts the Google Drive file ID from the CV upload URL included in the form response.

**Change Drive File Permissions** — calls the Drive API to set the file to `reader` access. Google Forms CV uploads are private by default. Importantly, this node's output connects to nothing downstream — it is a pure side-effect branch. Its only job is to make the file readable; the actual file ID for downloading comes from `Normalize Form Data` via a separate path. This means a Drive permissions failure does not block the main pipeline.

**Extract Resume File ID** — isolates the clean file ID for use in the download URL.

**Download File** — fetches the raw CV file (PDF, DOCX, or image) from Google Drive.

**Resume Parser API** — POSTs the file to the custom microservice at `https://api.inomadigital.com/parse-resume`. The service handles PDF, DOCX, and image inputs, extracts the text content, and normalizes whitespace, line breaks, and formatting. Returns clean plain text.

**Clean Resume Text** — final text cleanup within n8n before passing to the AI branch.

---

### 5. AI Branch (parallel)

Runs alongside the CV branch after normalization.

**Build Candidate Context** — assembles the structured candidate object: all normalized form fields combined into a single profile string ready for the prompt.

**Prepare Gemini Prompt** — constructs the full prompt. The prompt is strict about output format:

```javascript
const prompt = `
You are an AI recruitment assistant.
Analyze this candidate and return ONLY valid JSON.
DO NOT write explanations.
DO NOT write text.
RETURN JSON ONLY.

Required format:
{
  "score": number from 0-100,
  "strengths": ["strength1", "strength2"],
  "weaknesses": ["weakness1", "weakness2"],
  "recommendation": "Reject | Review | Interview"
}

Candidate profile:
${$json.candidate_profile}
`;
```

**Wait** — holds until the CV branch has completed and resume text is available, then the full context (form data + CV text) is passed to Gemini together.

**Gemini Evaluation** — HTTP POST to `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`. Authentication via `X-goog-api-key` header. Uses Google AI Studio (free tier). The node is configured with `retryOnFail: true`, `maxTries: 5`, `waitBetweenTries: 5000ms`, and `onError: continueRegularOutput` — if Gemini is unavailable after 5 attempts, the workflow continues and writes the candidate to Notion without an AI evaluation rather than failing the entire execution. This is the primary reason the workflow has maintained a 0% failure rate across 485 production executions.

**Parse JSON Response** — extracts and parses the JSON from Gemini's response. Includes fallback handling to strip any accidental markdown fencing before parsing.

**Store AI Results** — holds the parsed evaluation object for the merge step.

---

### 6. Merge Candidate Data

Combines the normalized form data with the parsed AI evaluation results into a single object for Notion write.

---

### 7. Create Notion Database Page

Creates the candidate record in the Notion hiring database. Sets top-level database properties: Name (title), Role, Status (defaults to "Applied"), Position Type, Work Type, Source, Email, WhatsApp, Date Applied, ID.

---

### 8. Get Notion Page ID

Retrieves the ID of the just-created page to use for subsequent block append operations. Notion's block append API requires the page ID separately from the database item creation response.

---

### 9. Append Basic Candidate Info

Appends the Basic Information section as a structured block to the Notion page body. Includes name, email, WhatsApp, city, source, role, position type, working mode, and application date.

---

### 10. Is Internship?

**Condition:** `position_type` equals `Internship`

Branches the remaining Notion page construction into two paths based on candidate type.

---

### 11a. Internship Path

**Internship Append** — appends the Internship Details section: student/graduate status, office availability, 3-month commitment, additional notes.

Then continues to Format AI Bullets → Append AI Evaluation.

---

### 11b. Full-time Path

**Append Role Heading** — adds the role-specific section header.

**Format Candidate Answers** — formats the role-specific screening Q&A into readable blocks.

**Append Job Questions** — appends Job Details section: availability, salary expectations, one-year commitment.

**Append Candidate Answers** — appends the formatted role screening answers.

Then continues to Format AI Bullets → Append AI Evaluation.

---

### 12. Format AI Bullets + Append AI Evaluation

Formats the Gemini output (score, strengths list, weaknesses list, recommendation) into Notion-native bullet blocks and appends the AI Evaluation section to the page.

---

### 13. Merge

Joins both branches (internship and full-time paths) back together. Execution ends here.

---

## Notion Page Structure

Each candidate lands as a full structured Notion document, not just a flat database row. This makes it usable as a working document during the review process — the team can add comments, update the Status field, and annotate without leaving Notion.

**Full-time candidate page:**
```
🧾 Basic Information
📎 Resume
💼 Job Details
⭕️ Role Questions
🤖 AI Candidate Evaluation
```

**Internship candidate page:**
```
🧾 Basic Information
📎 Resume
🎓 Internship Details
🤖 AI Candidate Evaluation
```

---

## Resume Parser Microservice

The custom parser at `api.inomadigital.com/parse-resume` is deployed on the same Hetzner VPS as n8n. It handles three input types:

- **PDF** — text extraction
- **DOCX** — document text extraction
- **Image** (JPG, PNG) — OCR-based text extraction

Output is normalized plain text with consistent spacing and line breaks, ready for inclusion in the Gemini prompt without token-wasting whitespace artifacts.

Keeping this as a separate microservice rather than a Code node in n8n means it can be updated, tested, or replaced independently of the workflow.

---

## Why n8n Over Alternatives?

| | n8n (self-hosted) | Zapier | Make |
|--|--|--|--|
| **Cost** | ~$5/mo VPS | $49+/mo | $9+/mo |
| **Custom JS nodes** | Yes | No | Limited |
| **Self-hostable** | Yes | No | No |
| **Data stays on your infra** | Yes | No | No |
| **Parallel branches** | Yes | No | Yes |
| **Execution logs** | Full detail | Basic | Good |

Data privacy was a deciding factor: candidate CVs and personal information stay on Inoma's own VPS, not on a third-party automation platform's servers.

---

## Production Notes

- n8n runs in Docker on Hetzner, volume-mounted for persistence across restarts
- Nginx handles SSL termination; Cloudflare proxies the public domain
- Gemini free tier handles current volume comfortably (tens of applications per role)
- The Wait nodes in both branches are a pragmatic solution to Google Drive's file processing delay and parallel branch timing — not elegant, but reliable in practice
- n8n's execution history provides a full audit trail of every application processed, with per-node input/output visible for debugging
