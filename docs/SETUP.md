# Setup Guide

Full instructions for deploying hirescout from scratch — self-hosting n8n, connecting all integrations, deploying the resume parser, and going live.

---

## 1. Self-Host n8n on a VPS

### Server Requirements

- Ubuntu 22.04 LTS
- Minimum: 2 vCPU, 4GB RAM (Hetzner CX21 or equivalent)
- A domain/subdomain pointed at the server (e.g. `n8n.yourdomain.com`)

### Install n8n with Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Create n8n data directory
mkdir -p /opt/n8n/data

# Run n8n
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v /opt/n8n/data:/home/node/.n8n \
  -e N8N_HOST=n8n.yourdomain.com \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=https \
  -e WEBHOOK_URL=https://n8n.yourdomain.com/ \
  n8nio/n8n
```

### Nginx Reverse Proxy + SSL

```bash
apt install -y nginx certbot python3-certbot-nginx

cat > /etc/nginx/sites-available/n8n << 'EOF'
server {
    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
EOF

ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d n8n.yourdomain.com
```

---

## 2. Deploy the Resume Parser Microservice

The workflow depends on a custom resume parsing service that accepts CV files and returns normalized plain text. You need to deploy this separately on the same or a different server.

The service should:
- Accept POST requests with a file upload
- Handle PDF, DOCX, and image (JPG/PNG) inputs
- Return extracted, normalized plain text

A minimal Node.js implementation using `pdf-parse`, `mammoth`, and `tesseract.js`:

```bash
mkdir resume-parser && cd resume-parser
npm init -y
npm install express multer pdf-parse mammoth tesseract.js
```

```javascript
// server.js
const express = require('express');
const multer = require('multer');
const pdfParse = require('pdf-parse');
const mammoth = require('mammoth');

const app = express();
const upload = multer({ storage: multer.memoryStorage() });

app.post('/parse-resume', upload.single('file'), async (req, res) => {
  const { mimetype, buffer } = req.file;

  try {
    let text = '';

    if (mimetype === 'application/pdf') {
      const result = await pdfParse(buffer);
      text = result.text;
    } else if (mimetype.includes('word') || mimetype.includes('docx')) {
      const result = await mammoth.extractRawText({ buffer });
      text = result.value;
    } else {
      return res.status(400).json({ error: 'Unsupported file type' });
    }

    // Normalize whitespace
    text = text.replace(/\s+/g, ' ').trim();
    res.json({ text });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => console.log('Resume parser running on :3000'));
```

Serve it behind Nginx at `api.yourdomain.com/parse-resume` with SSL, same pattern as n8n above.

---

## 3. Import the Workflow

1. Open your n8n instance
2. **Workflows → Import from File** → select `workflow.json`
3. All nodes import with structure intact — credentials will be unlinked (expected)

---

## 4. Configure Google Sheets Credential

1. In n8n: **Credentials → New → Google Sheets OAuth2 API**
2. Create a Google Cloud project at [console.cloud.google.com](https://console.cloud.google.com)
3. Enable **Google Sheets API** and **Google Drive API**
4. Create OAuth 2.0 credentials (Web Application type)
5. Add your n8n callback URL as an authorized redirect:
   `https://n8n.yourdomain.com/rest/oauth2-credential/callback`
6. Paste Client ID + Secret into n8n, authenticate

In the Google Sheets trigger node, set your **Spreadsheet ID** (from the Sheet URL) and the correct **Sheet Name**.

---

## 5. Configure Google Drive Credential

Use the same Google Cloud OAuth app as above. In n8n: **Credentials → New → Google Drive OAuth2 API**, authenticate with the same account that owns the Form/Sheet.

This credential is used by the Drive permission and download nodes.

---

## 6. Configure Gemini API

1. Get an API key at [aistudio.google.com](https://aistudio.google.com) (free tier)
2. In the Gemini Evaluation node, the key is passed via the `X-goog-api-key` header
3. Update the header value with your key
4. The endpoint is: `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent`

Update the model slug in the URL if you want to use a different Gemini version.

---

## 7. Configure Notion

1. Go to [notion.so/my-integrations](https://www.notion.so/my-integrations) → **New Integration**
2. Copy the Integration Token
3. In n8n: **Credentials → New → Notion API** → paste the token

**Set up the Notion database:**

Create a database with these properties:

| Property | Type |
|----------|------|
| Name | Title |
| Role | Select |
| Status | Select (Applied, Shortlisted, Reviewed, Interview, Rejected) |
| Position Type | Select (Full-time Job, Internship) |
| Work Type | Select (On-Site, Remote, Hybrid) |
| Source | Select (Facebook, Instagram, LinkedIn, WhatsApp Group, Referral, Other) |
| Email | Email |
| WhatsApp | Phone |
| Date Applied | Date |
| Rating /10 | Number |
| Notes | Text |
| Resume / CV | Files & Media |
| ID | Number (auto-increment) |

Connect the integration: open the database → **...** → **Add connections** → select your integration.

Copy the Database ID from the URL and update it in the **Create Notion Database Page** node.

---

## 8. Set Up the Google Forms Webhook

In your Google Sheet (linked to the Form), open **Extensions → Apps Script** and add:

```javascript
function onFormSubmit(e) {
  const webhookUrl = 'https://n8n.yourdomain.com/webhook/YOUR_WEBHOOK_PATH';

  const payload = {};
  for (const key in e.namedValues) {
    payload[key] = e.namedValues[key][0];
  }

  UrlFetchApp.fetch(webhookUrl, {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  });
}
```

Set this function as a trigger: **Triggers → Add Trigger → onFormSubmit → Form submit**.

Activate the workflow in n8n (toggle in the top bar). The webhook is live once the workflow is active.

---

## 9. Update Node References

After import, update these values across the workflow:

| Node | What to update |
|------|---------------|
| Webhook | Your webhook path |
| Google Sheets | Spreadsheet ID, Sheet name |
| Resume Parser API | Your microservice URL |
| Gemini Evaluation | `X-goog-api-key` header value, model URL |
| Create Notion DB Page | Database ID |
| All Notion append nodes | Confirm page ID reference is correct |

---

## 10. Test End-to-End

1. Submit a test application through your Google Form
2. Watch **Executions** tab in n8n — you should see the workflow trigger
3. Open the execution detail to see per-node input/output
4. Verify the candidate page appears in Notion with AI evaluation populated
5. Test both a Full-time and an Internship submission to confirm both Notion page structures build correctly

---

## Troubleshooting

**Webhook not triggering:** Confirm the workflow is active. Check the Apps Script trigger is set to "On form submit" not "On edit."

**Drive permission error:** The CV upload URL in the form response must be a valid Google Drive link. Ensure the Form is configured to upload files to Drive and the OAuth account has access to that Drive folder.

**Resume parser returning empty:** Check the microservice is running and accessible. Test it directly with a curl request before debugging the n8n node.

**Gemini returning non-JSON:** The model occasionally wraps output in markdown fences despite instructions. Add a cleanup step in the Parse JSON Response node to strip backtick fences before `JSON.parse()`.

**Notion write failing:** Confirm the integration is connected to the specific database (not just the workspace). The Database ID must be copied exactly from the URL.
