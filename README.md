# jd2resume_n8n

Paste a job description, get back a one-page tailored PDF resume. No UI, no accounts — just a webhook in and a PDF out.

The workflow runs entirely on self-hosted n8n. It calls the Anthropic API to extract JD metadata, rewrites each resume section to match the role and company archetype, validates character budgets, renders HTML, converts to PDF via Gotenberg, and confirms single-page output before returning the file path.

---

## Prerequisites

- Docker and Docker Compose
- Anthropic API key — [console.anthropic.com](https://console.anthropic.com)

---

## Setup

```bash
git clone https://github.com/your-username/jd2resume_n8n.git
cd jd2resume_n8n

cp .env.example .env
# open .env and paste your Anthropic API key

docker-compose up -d
```

The `output/` directory is created automatically on first run.

---

## Import the workflow into n8n

1. Open [http://localhost:5678](http://localhost:5678) in your browser
2. Default credentials on first launch: set up your owner account
3. Go to **Workflows → Import from file**
4. Select `workflow.json` from this repo
5. Click **Activate** (toggle in the top-right)

---

## Activate the webhook

After activating the workflow, the webhook URL is:

```
http://localhost:5678/webhook/resume-tailor
```

---

## Send a job description

```bash
curl -X POST http://localhost:5678/webhook/resume-tailor \
  -H "Content-Type: application/json" \
  -d '{
    "jd": "We are hiring a Senior Data Scientist to lead experimentation and build ML models for our growth team at Acme Corp..."
  }'
```

Successful response:

```json
{
  "status": "success",
  "file_path": "/data/output/Sridhar_Resume_Acme_Corp_Senior_Data_Scientist.pdf",
  "company": "Acme Corp",
  "role": "Senior Data Scientist"
}
```

---

## Where to find the PDF

PDFs are saved to the `output/` directory in this repo (mounted into the container). The filename pattern is:

```
output/Sridhar_Resume_{Company}_{Role}.pdf
```

---

## Update the master resume

Edit `data/master_resume.json` directly. No restart needed — the workflow reads it on every request.

Fields you might update:
- `summary` — baseline summary (gets rewritten per JD anyway)
- `experience[].bullets` — source bullets the LLM rewrites from
- `tech_stack` — full list; the workflow selects the 8-12 most relevant per JD
- `projects` — add new projects here

---

## How the workflow handles length validation

After tailoring, the workflow checks:

| Field | Limit |
|---|---|
| Summary | 280 chars |
| Each bullet | 150–220 chars |
| Tech stack items | 8–12 |
| Project description | 180 chars each |

If any field is out of bounds, it retries the Anthropic call with compression instructions (max 2 retries). After PDF generation, it also verifies the page count via pdf-lib. If the PDF is more than one page, it loops back with tighter budgets (max 2 additional retries).

---

## Troubleshooting

**API errors (401 / 403)**
Check that `ANTHROPIC_API_KEY` in `.env` is valid and has credits. Restart the stack after editing `.env`:
```bash
docker-compose down && docker-compose up -d
```

**Gotenberg connection refused**
Make sure both containers started:
```bash
docker-compose ps
```
The n8n container reaches Gotenberg at `http://gotenberg:3000` over the internal `resume_net` network. If Gotenberg is not listed as running, check logs:
```bash
docker-compose logs gotenberg
```

**Page overflow loop**
If the workflow keeps looping on page count, the content is likely too long for the CSS budget. You can tighten the budgets in the `Validate Lengths` node (lines with `BULLET_CEIL`, `SUMMARY_LIMIT`) or reduce the number of bullets per job in `master_resume.json`.

**pdf-lib not found**
The `NODE_FUNCTION_ALLOW_EXTERNAL=pdf-lib` env var is set in `docker-compose.yml`. If the Code node throws a module error, confirm that env var made it into the container:
```bash
docker-compose exec n8n env | grep pdf
```

---

## Project structure

```
jd2resume_n8n/
├── workflow.json          # n8n workflow — import this
├── data/
│   └── master_resume.json # source resume data
├── templates/
│   └── resume.html        # HTML template with {{placeholders}}
├── prompts/
│   ├── tailor_system.md   # system prompt for resume tailoring
│   └── extract_metadata.md # prompt to extract company + role from JD
├── output/                # generated PDFs land here
├── docker-compose.yml
├── .env.example
└── README.md
```
