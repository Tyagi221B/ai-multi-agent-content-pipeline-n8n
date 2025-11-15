# 🚀 AI Content Automation Pipeline
### n8n + OpenAI + Google Vertex AI VEO + Drive

> **Assignment submission for AI Automation Engineer role**

This workflow automatically discovers trending AI topics from Google News and YouTube, uses GPT-5 to select the most relevant ones, generates professional blog posts, creates cinematic videos using Google's VEO 3.1 model, uploads everything to Google Drive, logs to Sheets, and sends Slack notifications.

**This was my first time building something this complex in n8n — and I went all-in.**

---

## 📹 Demo & Documentation

- **Loom Walkthrough:** [Loom Video](https://www.loom.com/share/93377041293a4fde809a3f5f8e06c047)
  *(Explains every agent, node, bug fix, and design decision)*

- **Live Assignment Sheet:** [Google Sheet](https://docs.google.com/spreadsheets/d/1BFNdg2be6sCb2nX2dAtQgmmm_ecU8ol8LA2_AyLnMhg/edit?usp=sharing)
  *(Real-time log of all generated content)*

---

## 🏗️ High-Level Architecture

```
Google News RSS + YouTube API
         ↓
Extract & Normalize Titles
         ↓
Deduplicate + Score Topics
         ↓
GPT-5 Ranks & Selects Best 5
         ↓
Generate Blog + Video Prompts (GPT-4o)
         ↓
┌────────────────────┬────────────────────┐
│  Blog Generation   │  Video Generation  │
│  (GPT-3.5-turbo)   │  (Google VEO 3.1)  │
└────────────────────┴────────────────────┘
         ↓                    ↓
    Upload .md           Poll Operation
         ↓                    ↓
   Google Drive          Download Video
         ↓                    ↓
    Get Link             Upload to Drive
         └────────┬───────────┘
                  ↓
         Merge Blog + Video Links
                  ↓
         Append to Google Sheets
                  ↓
         Send Slack Notification
```

**Runs automatically every day at 9 AM.**

---

## 🧩 Pipeline Stages

### **1. Topic Discovery Agent**
- Fetches trending news from Google News RSS feed
- Pulls top videos from YouTube API
- Extracts clean titles from both sources

**Nodes:** `HTTP Request`, `Fetch YouTube AI Videos`, `Trend Extract Titles`, `YT Extract Titles`

### **2. Normalization & Deduplication Agent**
- Removes HTML tags, emojis, special chars
- Normalizes whitespace and casing
- Detects near-duplicates using string similarity
- Scores by frequency and source diversity

**Nodes:** `Merge`, `Normalize & Dedupe Topics`, `Batch Topics`

### **3. Topic Selection Agent (GPT-5)**
- Receives 20 candidate topics
- Ranks by:
  - Relevance to AI automation
  - Freshness and trending potential
  - Content creation viability
- Ensures minimum 2 YouTube sources
- Returns **exactly 5 final topics**

**Nodes:** `Filter & Rank Best Topics`, `Unwrap Final Topics`

### **4. Prompt Generation Agent (GPT-4o)**
- For each selected topic, generates:
  - **Blog Prompt:** Brief, outline, keywords, tone, word count
  - **Video Prompt:** Scene description, duration, VEO instructions
  - **Tags:** Content categories

**Nodes:** `Message a model`, `Parse Prompt JSON`

### **5. Content Generation Agents**
**Blog:**
- GPT-3.5-turbo writes 900-1100 word posts
- Includes TL;DR, headings, code snippets
- Formatted as clean markdown
- Uploaded to Google Drive

**Video:**
- Builds VEO 3.1 API request payload
- Submits long-running video generation job
- Stores output in **Google Cloud Storage** (GCS)
- Polls until completion (retry loop with 35s delays)
- Downloads video and uploads to Drive

**Nodes:** `Generate Blog Post`, `Clean Blog Output`, `ConvertToBinary`, `Upload file`, `prepare VEO payload`, `Get Token`, `Request VEO Video`, `Wait`, `Poll VEO Operation`, `If`, `Extract GCS URI`, `Download Video`, `Upload Video`

### **6. Metadata Aggregation Agent**
- Normalizes filenames and topics
- Extracts shareable Drive links
- Merges blog + video metadata by index

**Nodes:** `AttachBlogLink`, `Extract Video URL From Drive`, `Metadata Normalizer`, `Merge2`, `Merge3`

### **7. Logging & Notification Agent**
- Formats rows for Google Sheets
- Appends:
  - Topic
  - Blog link
  - Video link
  - Prompts used
  - Timestamp
  - Status
- Sends rich Slack message with all metadata

**Nodes:** `PrepareSheetRows`, `Append row in sheet`, `Slack Notification`

---

## 🔥 Edge Cases & Solutions

### **1. VEO Base64 Overflow**
**Problem:** VEO poll initially returned entire video as base64 → n8n memory crash
**Fix:** Changed VEO API to use `storageUri` → video saved to **GCS bucket** → only fetch URL

### **2. Google Cloud Token Rotation**
**Problem:** OAuth tokens expired mid-execution → pipeline failures
**Fix:** Built a lightweight Flask server on Hostinger that refreshes tokens via service account → `/token` endpoint protected by API key

### **3. Batch Processing Merges**
**Problem:** n8n splits arrays into separate items → merge nodes misalign
**Fix:** Wrapped topic arrays into single objects → controlled fan-out with index tracking

### **4. Filename Sanitization**
**Problem:** Raw titles contain `/`, `:`, emojis → Drive upload errors
**Fix:** Sanitize function removes illegal chars, caps length at 200, replaces with underscores

### **5. VEO Polling Race Condition**
**Problem:** Sometimes `done: false` but job already completed
**Fix:** Retry loop with max 10 iterations, 35s delay, fallback error handling

### **6. JSON Parsing from LLM Outputs**
**Problem:** GPT sometimes wraps JSON in markdown code blocks or escapes quotes
**Fix:** Iterative unescape + regex extraction function in `Parse Prompt JSON` node

---

## 📊 Scaling to 100+ Posts Per Day

Current system handles **5 items smoothly**. To scale to **100/day**:

### **1. Queue-Based Architecture**
- Replace sequential processing with Redis/Pub/Sub
- Push 100 tasks to queue
- n8n workers process in parallel (10-20 concurrent)

### **2. State Management**
- Move data to Firestore/Supabase
- n8n only orchestrates, doesn't store payloads
- Enables resume on failure

### **3. Event-Driven VEO Polling**
- Replace polling loop with Cloud Pub/Sub
- VEO publishes completion event → triggers n8n webhook
- Zero wasted compute

### **4. Microservice for Heavy Lifting**
- Deploy Cloud Run service for:
  - Filename normalization
  - Prompt building
  - Video download/upload
- n8n calls service via HTTP
- Better error isolation

### **5. Batch Drive Uploads**
- Use Drive API batch requests
- Upload 100 files in single call
- 10x faster than sequential

### **6. Cost Optimization**
- Cache trending topics for 6 hours
- Use GPT-3.5-turbo for ranking (cheaper than GPT-5)
- Store VEO videos in Cloudflare R2 (cheaper than GCS)

**Result:** Same pipeline, 20x throughput, 50% lower cost.

---

## 🛠️ Setup Instructions

### **Prerequisites**
- n8n (self-hosted or cloud)
- Google Cloud project with:
  - Vertex AI (VEO 3.1 access)
  - Cloud Storage bucket
  - Service account with appropriate permissions
- Google Drive API enabled
- Google Sheets API enabled
- OpenAI API key
- Slack incoming webhook URL

### **Installation**

1. **Clone and import workflow:**
   ```bash
   # Import FINAL1_Assignment.json into n8n
   ```

2. **Set up credentials:**
   - **OpenAI:** Add API key in n8n credentials
   - **Google Drive OAuth2:** Configure OAuth client
   - **Google Sheets OAuth2:** Same as Drive
   - **Google Cloud Token Server:** Update the token endpoint URL and API key

3. **Update environment variables in the workflow:**
   - Google Cloud project ID 
   - GCS bucket path 
   - Token server endpoint
   - Slack webhook URL 
   - Google Sheet ID 

4. **Test the workflow:**
   ```bash
   # Click "Execute Workflow" in n8n
   # Monitor execution logs
   ```

5. **Activate scheduler:**
   - Enable the workflow
   - Confirm trigger time (default: 9 AM daily)

---

## 📁 Repository Contents

```
.
├── FINAL1_Assignment.json     # Complete n8n workflow
├── ASSIGNMENT_README.md        # This file
├── docker-compose.yml          # Docker setup for n8n
└── .env                        # Environment variables (not committed)
```

---

## 🧪 Testing

**Manual Test:**
```bash
# In n8n, click "Execute Workflow"
# Expected outputs:
# - 5 blog .md files in Google Drive
# - 5 video .mp4 files in Google Drive
# - 5 rows in Google Sheet
# - 5 Slack notifications
```

**Monitoring:**
- Check n8n execution logs
- Verify Google Sheet for new rows
- Confirm Slack channel for notifications
- Inspect Drive folder for uploads

---

## 🐛 Known Limitations

1. **VEO Rate Limits:** Max 10 concurrent video jobs → sequential processing
2. **Token Refresh:** Requires external server (could be replaced with n8n-native refresh)
3. **Error Recovery:** Partial failures don't resume mid-batch
4. **Cost:** Running 5 posts/day costs ~$2-3 (mostly GPT + VEO)

---

## 🎯 Key Achievements

- ✅ Fully autonomous content pipeline
- ✅ Multi-source data aggregation (News + YouTube)
- ✅ AI-powered topic ranking and filtering
- ✅ Blog + video generation in one flow
- ✅ Cloud storage integration (GCS + Drive)
- ✅ Comprehensive logging and notifications
- ✅ Robust error handling and retries
- ✅ First major n8n workflow (learned on the job)

---

## 🙏 Final Notes

This assignment pushed me to:
- Master async long-running operations
- Handle OAuth token lifecycles
- Integrate Google Cloud Vertex AI
- Build complex merge/split logic in n8n
- Debug JSON parsing from LLM outputs
- Design for scalability from day one

I loved every minute of this challenge.

**For questions or production deployment inquiries, reach out!**

---

**Built with ❤️ using n8n, OpenAI, and Google Cloud**
# ai-multi-agent-content-pipeline-n8n
