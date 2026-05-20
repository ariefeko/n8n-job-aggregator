# 🤖 Job Aggregator — Automated Daily Job Digest via Telegram

An automated workflow built with **n8n** that aggregates remote job listings from multiple job boards, filters by tech stack keywords, summarizes each job using **Gemini AI**, and delivers a daily digest straight to **Telegram**.

No more manually checking job boards every day — let the automation do it for you.

---

## ✨ Features

- 📡 **Multi-source aggregation** — Fetches jobs from Arbeitnow and Remotive simultaneously
- 🔍 **Smart filtering** — Filters by tech stack keywords (Laravel, PHP, Node.js, Vue.js, Fullstack, etc.)
- 🤖 **AI-powered summary** — Each job is summarized in Bahasa Indonesia using Google Gemini AI
- 📱 **Telegram delivery** — Daily digest sent directly to your Telegram every morning at 8 AM
- ⚡ **Fully automated** — Runs on schedule, zero manual effort after setup

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Automation Engine | n8n (self-hosted via Docker) |
| Job Source 1 | Arbeitnow API (free, no auth) |
| Job Source 2 | Remotive API (free, no auth) |
| AI Summarizer | Google Gemini API (free tier) |
| Notification | Telegram Bot API |
| Runtime | Docker |

---

## 🔄 Workflow Architecture

```
Schedule Trigger (08:00 AM daily)
         ↓              ↓
   Arbeitnow API    Remotive API
         ↘              ↙
      Code (merge + filter by keywords)
                  ↓
          Gemini AI (summarize)
                  ↓
           Telegram Bot
                  ↓
      📱 Daily digest on your phone
```

---

## 📱 Sample Output

```
🔍 Job Digest Harian
📅 Senin, 19 Mei 2026

💼 Senior Laravel Developer
🏢 Acme Corp
📍 Remote - Worldwide
🏷️ laravel, php, backend, mysql, rest-api
🟢 Remotive

🤖 AI Summary:
Posisi ini cocok untuk developer Laravel senior yang berpengalaman
membangun REST API skala besar. Perusahaan menawarkan full remote
dengan tim yang tersebar di berbagai negara.

🔗 https://remotive.com/remote-jobs/...
```

---

## 🚀 Getting Started

### Requirements

- Docker installed
- Telegram account
- Google account (for Gemini API key)

---

### Step 1 — Run n8n via Docker

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

Open browser: `http://localhost:5678`

---

### Step 2 — Setup Telegram Bot

1. Open Telegram → search **@BotFather**
2. Send `/newbot` → follow instructions
3. Copy the **Bot Token**
4. Search **@userinfobot** → get your **Chat ID**

---

### Step 3 — Get Gemini API Key

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Click **Get API Key** → **Create API Key**
3. Copy the key — keep it safe, never share it

---

### Step 4 — Import & Configure Workflow

1. In n8n, create a new workflow
2. Add nodes in this order:
   - **Schedule Trigger** → set to daily at 8:00 AM
   - **HTTP Request** (Arbeitnow) → `GET https://arbeitnow.com/api/job-board-api`
   - **HTTP Request** (Remotive) → `GET https://remotive.com/api/remote-jobs?category=software-dev`
   - **Code** → merge + filter logic (see below)
   - **HTTP Request** (Gemini) → POST to Gemini API
   - **Telegram** → send digest message

---

### Filter & Merge Code

Paste this in the **Code** node:

```javascript
const allInputs = $input.all();
const keywords = ['laravel', 'php', 'backend', 'node', 'fullstack', 'full-stack', 'vue'];

let allJobs = [];

for (const input of allInputs) {
  const data = input.json;

  // Arbeitnow
  if (data.data && Array.isArray(data.data)) {
    const jobs = data.data.map(job => ({
      title: job.title,
      company: job.company_name,
      location: job.location || 'Remote',
      url: `https://arbeitnow.com/jobs/${job.slug}`,
      tags: job.tags?.join(', ') || '',
      source: '🌐 Arbeitnow',
    }));
    allJobs = allJobs.concat(jobs);
  }

  // Remotive
  if (data.jobs && Array.isArray(data.jobs)) {
    const jobs = data.jobs.map(job => ({
      title: job.title,
      company: job.company_name,
      location: job.candidate_required_location || 'Remote',
      url: job.url,
      tags: job.tags?.join(', ') || '',
      source: '🟢 Remotive',
    }));
    allJobs = allJobs.concat(jobs);
  }
}

const filtered = allJobs.filter(job => {
  const text = `${job.title} ${job.tags}`.toLowerCase();
  return keywords.some(k => text.includes(k));
});

return filtered.slice(0, 3).map(r => ({ json: r }));
```

---

### Gemini API Node Setup

- **Method:** POST
- **URL:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=YOUR_API_KEY`
- **Header:** `Content-Type: application/json`
- **Body:**

```json
{
  "contents": [
    {
      "parts": [
        {
          "text": "Summarize this job in 2 sentences in Bahasa Indonesia, focus on what makes it interesting for a senior Laravel/backend developer:\n\nTitle: {{ $json.title }}\nCompany: {{ $json.company }}\nLocation: {{ $json.location }}\nTags: {{ $json.tags }}\nSource: {{ $json.source }}"
        }
      ]
    }
  ]
}
```

---

### Telegram Message Template

```
🔍 *Job Digest Harian*
📅 {{ new Date().toLocaleDateString('id-ID', {weekday:'long', year:'numeric', month:'long', day:'numeric'}) }}

💼 *{{ $('Code in JavaScript').item.json.title }}*
🏢 {{ $('Code in JavaScript').item.json.company }}
📍 {{ $('Code in JavaScript').item.json.location }}
🏷️ {{ $('Code in JavaScript').item.json.tags }}
{{ $('Code in JavaScript').item.json.source }}

🤖 *AI Summary:*
{{ $json.candidates[0].content.parts[0].text }}

🔗 {{ $('Code in JavaScript').item.json.url }}
```

---

## 🗺️ Roadmap

- [ ] Add more job sources (Jobicy, We Work Remotely)
- [ ] Filter by salary range
- [ ] Save jobs to Google Sheets / Notion
- [ ] Avoid duplicate jobs across runs
- [ ] Deploy to VPS for 24/7 operation
- [ ] Add "Apply" tracking — mark jobs as applied

---

## 💡 Why I Built This

Manually checking multiple job boards daily is time-consuming, especially when you're actively job hunting. This automation saves ~30 minutes daily and ensures I never miss a relevant remote opportunity.

Built as part of my automation portfolio to demonstrate practical n8n + AI integration skills.

---

## 👨‍💻 Author

**Arief Eko Wicaksono**
Senior Fullstack Engineer — Laravel · Node.js · Vue.js · n8n Automation

- 🌐 [LinkedIn](https://www.linkedin.com/in/arief-eko-wicaksono-4175a12a)
- 💻 [GitHub](https://github.com/ariefeko)
- 📧 riff.sense@gmail.com

---

## 📄 License

MIT License — feel free to fork and customize for your own job search!
