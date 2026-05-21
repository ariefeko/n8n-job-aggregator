# 🤖 Job Aggregator — AI-Powered Daily Job Digest

An automated workflow built with **n8n** that aggregates remote job listings from multiple job boards, filters by tech stack, analyzes match percentage using **Groq AI**, and delivers a daily digest straight to **Telegram**.

No more manually checking job boards every day — let the automation do it for you.

---

## ✨ Features

- **Multi-source aggregation** — Fetches jobs from 4 job boards simultaneously (Arbeitnow, Remotive, Jobicy, Himalayas)
- **Smart filtering** — Filters by tech stack keywords, rejects irrelevant tech, blocks onsite/restricted jobs
- **Location-aware** — Only shows remote-friendly jobs, skips country-restricted listings
- **AI-powered analysis** — Each job analyzed by Groq AI with match %, summary, strengths, gaps, and verdict
- **Telegram delivery** — Daily digest sent directly to Telegram every morning at 8 AM
- **Fully automated** — Runs on schedule, zero manual effort after setup
- **Self-hosted** — Runs locally via Docker, zero cloud cost

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Automation Engine | n8n (self-hosted via Docker) |
| Job Source 1 | Arbeitnow API (free, no auth) |
| Job Source 2 | Remotive API (free, no auth) |
| Job Source 3 | Jobicy API (free, no auth) |
| Job Source 4 | Himalayas API (free, no auth) |
| AI Analyzer | Groq API — llama-3.3-70b-versatile (free tier) |
| Notification | Telegram Bot API |
| Runtime | Docker |

---

## 🔄 Workflow Architecture

```
Schedule Trigger (08:00 AM daily)
    ↓         ↓         ↓         ↓
Arbeitnow  Remotive  Jobicy  Himalayas
    ↘         ↙         ↘       ↙
       Code (merge + filter + dedupe)
                    ↓
             Groq AI (analyze)
                    ↓
              Telegram Bot
                    ↓
         📱 Daily digest on your phone
```

---

## 📱 Sample Output

```
💼 Senior Laravel Backend Engineer
🏢 acme-corp
📍 Remote
🏷️ Software Engineering
🌐 Arbeitnow

MATCH: 85%
SUMMARY: Strong fit for a senior Laravel engineer with REST API and CI/CD experience.
STRENGTHS: Laravel, Node.js, CI/CD
GAPS: No significant gaps
VERDICT: APPLY NOW ✅

🔗 https://arbeitnow.com/jobs/senior-laravel-...
```

---

## 🚀 Getting Started

### Requirements

- Docker installed
- Telegram account
- Groq account (free) — [console.groq.com](https://console.groq.com)

---

### Step 1 — Run n8n via Docker

```bash
docker run -d \
  --name n8n \
  --restart unless-stopped \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  --dns 10.0.3.1 \
  n8nio/n8n
```

Open browser: `http://localhost:5678`

> **Note:** If you have DNS issues inside Docker, use `--dns 10.0.3.1` (dnsmasq) or your local DNS resolver.

---

### Step 2 — Setup Telegram Bot

1. Open Telegram → search **@BotFather**
2. Send `/newbot` → follow instructions
3. Copy the **Bot Token**
4. Search **@userinfobot** → get your **Chat ID**
5. Open the bot and send `/start` before first use

---

### Step 3 — Get Groq API Key

1. Go to [console.groq.com](https://console.groq.com)
2. Sign in with Google
3. Click **API Keys** → **Create API Key**
4. Copy the key — keep it safe, never share it

---

### Step 4 — Configure Workflow Nodes

#### HTTP Request Nodes (4 sources)

| Source | Method | URL |
|---|---|---|
| Arbeitnow | GET | `https://arbeitnow.com/api/job-board-api` |
| Remotive | GET | `https://remotive.com/api/remote-jobs?search=laravel` |
| Jobicy | GET | `https://jobicy.com/api/v2/remote-jobs?count=20&tag=laravel` |
| Himalayas | GET | `https://himalayas.app/jobs/api/search?q=laravel&sort=recent` |

Connect all 4 HTTP Request nodes → Code node.

---

### Filter & Merge Code

Paste this in the **Code** node:

```javascript
const allInputs = $input.all();

// TECH KEYWORDS — must match at least one
const techKeywords = [
  'laravel', 'php', 'node', 'node.js',
  'react', 'backend', 'fullstack',
  'full-stack', 'full stack', 'inertia'
];

// HARD REJECT TECH
const rejectTech = [
  'ruby', 'rails',
  'java ', 'spring boot', 'kotlin',
  'ios', 'swift', 'android', 'flutter',
  'django', 'flask', '.net', 'c#',
  'golang', 'go developer', 'rust',
  'wordpress', 'shopify',
  'devops engineer', 'sre ', 'site reliability'
];

// HARD REJECT CONDITIONS
const rejectKeywords = [
  'junior', 'intern', 'graduate',
  'onsite', 'on-site', 'office-based',
  'must live in', 'must be based',
  'local candidates', 'only candidates',
  'citizen only', 'us citizen', 'eu citizen',
  'no sponsorship', 'eligible to work',
  'work authorization', 'permanent resident'
];

let allJobs = [];

for (const input of allInputs) {
  const data = input.json;

  // Arbeitnow
  if (data.data && Array.isArray(data.data)) {
    const jobs = data.data.map(job => ({
      title: job.title || '',
      company: job.company_name || '',
      location: job.location || '',
      url: `https://arbeitnow.com/jobs/${job.slug}`,
      tags: job.tags?.join(', ') || '',
      description: (job.description || '').replace(/<[^>]*>/g, '').substring(0, 500),
      source: '🌐 Arbeitnow',
    }));
    allJobs = allJobs.concat(jobs);
  }

  // Remotive
  if (data.jobs && Array.isArray(data.jobs) && data.jobs[0]?.candidate_required_location !== undefined) {
    const jobs = data.jobs.map(job => ({
      title: job.title || '',
      company: job.company_name || '',
      location: job.candidate_required_location || '',
      url: job.url || '',
      tags: job.tags?.join(', ') || '',
      description: (job.description || '').replace(/<[^>]*>/g, '').substring(0, 500),
      source: '🟢 Remotive',
    }));
    allJobs = allJobs.concat(jobs);
  }

  // Jobicy
  if (data.jobs && Array.isArray(data.jobs) && data.jobs[0]?.jobSlug !== undefined) {
    const jobs = data.jobs.map(job => ({
      title: job.jobTitle || '',
      company: job.companyName || '',
      location: job.jobGeo || '',
      url: job.url || '',
      tags: job.jobIndustry?.join(', ') || '',
      description: (job.jobDescription || job.jobExcerpt || '').replace(/<[^>]*>/g, '').substring(0, 500),
      source: '🔵 Jobicy',
    }));
    allJobs = allJobs.concat(jobs);
  }

  // Himalayas
  if (data.jobs && Array.isArray(data.jobs) && data.jobs[0]?.companySlug !== undefined) {
    const jobs = data.jobs.map(job => ({
      title: job.title || '',
      company: job.companySlug || '',
      location: job.locationRestrictions?.join(', ') || '',
      url: job.applicationLink || '',
      tags: job.categories?.join(', ') || '',
      description: (job.description || job.excerpt || '').replace(/<[^>]*>/g, '').substring(0, 500),
      source: '🏔️ Himalayas',
    }));
    allJobs = allJobs.concat(jobs);
  }
}

// DEDUPE + FILTER
const seen = new Set();

const filtered = allJobs
  .filter(job => {
    const titleLower = job.title.toLowerCase();
    const tagsLower = job.tags.toLowerCase();
    const descLower = job.description.toLowerCase();
    const locationLower = job.location.toLowerCase().trim();
    const fullText = `${titleLower} ${tagsLower} ${descLower}`;
    const titleAndTags = `${titleLower} ${tagsLower}`;

    // 1. Must match at least one tech keyword
    const techMatch = techKeywords.some(k => fullText.includes(k));
    if (!techMatch) return false;

    // 2. Reject if title or tags dominant another tech
    const techRejected = rejectTech.some(k => titleAndTags.includes(k));
    if (techRejected) return false;

    // 3. Reject other conditions
    const condRejected = rejectKeywords.some(k => fullText.includes(k));
    if (condRejected) return false;

    // 4. Location filter
    // Empty → allow | has "remote" → allow | no "remote" → skip
    if (locationLower && !locationLower.includes('remote')) return false;

    return true;
  })
  // 5. Dedupe
  .filter(job => {
    const key = (job.company + job.title + job.location).toLowerCase();
    if (seen.has(key)) return false;
    seen.add(key);
    return true;
  })
  .slice(0, 10);

return filtered.map(job => ({ json: job }));
```

---

### Groq AI Node Setup

- **Method:** POST
- **URL:** `https://api.groq.com/openai/v1/chat/completions`
- **Headers:**
  - `Content-Type: application/json`
  - `Authorization: Bearer YOUR_GROQ_API_KEY`
- **Body:**

```json
{
  "model": "llama-3.3-70b-versatile",
  "messages": [
    {
      "role": "system",
      "content": "You are a strategic global job match analyzer."
    },
    {
      "role": "user",
      "content": "Candidate:\n- 9+ years Software Engineer\n- Laravel, Node.js, Vue.js, React, REST API, CI/CD, Docker, MySQL, PostgreSQL, Redis\n- Strong backend, integration, queue, cron, reliability and enterprise systems\n- Experience with BRI banking and Nestle projects\n- Growing React/fullstack direction\n- Prefer remote or sustainable hybrid\n\nScoring weights:\nTechnical match 40%\nRemote/hybrid sustainability 20%\nSeniority fit 15%\nLocation/timezone feasibility 10%\nCareer growth 10%\nEnterprise/reliability relevance 5%\n\nJob:\nTitle: {{ $json.title }}\nCompany: {{ $json.company }}\nLocation: {{ $json.location }}\nTags: {{ $json.tags }}\n\nRespond ONLY in English. Maximum 5 lines.\n\nFormat:\nMATCH: [0-100]%\nSUMMARY: [1 sentence]\nSTRENGTHS: [top 3]\nGAPS: [gap or No significant gaps]\nVERDICT: APPLY NOW ✅ / CONSIDER 🤔 / SKIP ❌"
    }
  ],
  "temperature": 0.2,
  "max_tokens": 250
}
```

---

### Telegram Message Template

```
💼 {{ $('Code in JavaScript').item.json.title }}
🏢 {{ $('Code in JavaScript').item.json.company }}
📍 {{ $('Code in JavaScript').item.json.location }}
🏷️ {{ $('Code in JavaScript').item.json.tags }}
{{ $('Code in JavaScript').item.json.source }}

{{ $json.choices[0].message.content }}

🔗 {{ $('Code in JavaScript').item.json.url }}
```

---

## 🗺️ Roadmap

- [x] Multi-source aggregation (4 job boards)
- [x] Smart keyword filtering
- [x] Location-aware filtering (remote only)
- [x] AI-powered match analysis with verdict
- [x] Deduplication across sources
- [ ] WhatsApp delivery via CallMeBot
- [ ] Save jobs to Google Sheets / Notion
- [ ] Avoid duplicate jobs across runs (persistent storage)
- [ ] Deploy to VPS for 24/7 operation
- [ ] Add "Apply" tracking — mark jobs as applied

---

## 💡 Why I Built This

Manually checking multiple job boards daily is time-consuming, especially when you're actively job hunting. This automation saves ~30 minutes daily, ensures I never miss a relevant remote opportunity, and uses AI to pre-qualify each job before I even open it.

Built as part of my automation portfolio to demonstrate practical n8n + AI integration skills.

---

## 👨‍💻 Author

**Arief Eko Wicaksono**
Fullstack Engineer — Laravel · Node.js · Vue.js · n8n Automation

- 🌐 [Portfolio](https://ariefeko.github.io)
- 💼 [LinkedIn](https://www.linkedin.com/in/arief-eko-wicaksono-4175a12a)
- 💻 [GitHub](https://github.com/ariefeko)
- 📧 riff.sense@gmail.com
