# FreightQuote AI — Milestone 3

**Infosys Springboard Internship 7.0 · Batch 4 · Milestone 3**
Team / author: [your name] · Mentor: [mentor name]

FreightQuote AI is an agentic quoting and fleet-watch platform for sea freight. A team of seven agents builds every quote (route, weather, price, customs, margin, documents, alerts), five fleet-watch agents monitor vessels every 60 seconds, and an AI Copilot answers questions with exact SQL numbers and cited knowledge-base passages. Agents propose, people approve, everything is audited.

**Stack:** FastAPI · SQLite · LangGraph · scikit-learn · RAG (FAISS + BM25 + reranker) · Qwen on a Colab T4 GPU

---

## 1. Repository contents (`Milestone3/`)

| File | Purpose |
|---|---|
| `FreightQuoteAI_RAG_Builder_FINAL.ipynb` | Crawls websites and PDFs and builds the knowledge base. **Run first.** |
| `FreightQuote_AI_Final_Code.ipynb` | The complete app: writes all modules, starts the server, prints the public link. |
| `taught_questions.csv` | 67 questions taught to the Copilot (`question, answer, keywords`). |
| `FreightQuote_AI_Milestone3.pptx` / `.pdf` | 5-minute presentation, 6 slides. |
| `screenshots/` | Evidence screenshots listed in section 8. |

Notebook outputs are cleared. No keys, tokens or `.db` files are stored in the repository.

---

## 2. How to run (Google Colab)

### A. Build the knowledge base (once)
1. Open `FreightQuoteAI_RAG_Builder_FINAL.ipynb` in Colab and choose **Runtime → Change runtime type → T4 GPU**.
2. **Runtime → Run all.** Allow Google Drive access when asked (the first cell mounts Drive before the crawl, so PDFs survive a runtime reset).
3. Wait for the crawl (265 seed websites, depth 3, up to 100 PDFs per seed). If Colab disconnects, run all again: downloaded files are reused.
4. The last cell prints the number of files, documents and chunks and checks the 150+ target.
5. Result in Drive: `MyDrive/FreightQuote_AI/` with `rag_pdfs/`, `faiss_indexes/`, `bm25_indexes/` and `synthetic_kb/all_chunks.jsonl`.

### B. Start the app
1. Open `FreightQuote_AI_Final_Code.ipynb`, choose **T4 GPU**.
2. Keep `USE_DRIVE = True` and `PERSIST = True` (chats, taught answers and data are backed up to Drive).
3. Optional: add the Colab secret `NGROK_AUTHTOKEN` if the normal link is blocked.
4. **Runtime → Run all**, allow Drive access, and open the printed link.

### C. First checks
1. Sign in as Admin → **Admin → System check**: "Knowledge base (RAG)" must show your chunks.
2. **Admin → Teach Copilot** → import `taught_questions.csv`.
3. Ask 10 knowledge questions: answers are short and numbered with `[1] [2]` and a Sources list.

---

## 3. Roles and access

| Role | Can do |
|---|---|
| Admin | Everything, plus autonomy policy, approval chains, System check, Copilot insights, Teach Copilot |
| Operations Manager | Approve re-routes, quote holds, margin reviews; run agent cycles |
| Freight Broker | Handle quotes and customs documents, resolve tickets |
| Auditor | Read-only view of everything, including the audit trail |
| Customer | Request quotes and see the result |

Demo sign-ins (demo mode only) are listed on the login screen. A role without the privilege receives **HTTP 403** (for example a Customer opening an Admin page). Sign-in uses JWT in an HTTP-only cookie, PBKDF2-SHA256 password hashing, rate limits and a four-eyes approval rule. All timestamps are IST.

---

## 4. How the agents decide

- **Quote pipeline (LangGraph):** route → weather → (if weather is High, re-plan the route avoiding Suez and the Red Sea) → pricing → customs → margin → documents → alerts.
- **Risk** = 40% weather + 30% customs + 30% congestion.
- **Decision:** *On Hold* if customs documents are missing or risk > 0.7; *Pending Approval* if price > $50,000 or margin < 10%; otherwise *Approved*.
- **Fleet watch:** five agents (route, weather, customs, quote, IsolationForest anomaly) run every 60 seconds and propose actions that wait for approval or run automatically per the autonomy policy.
- **Price** comes from a Random Forest model plus fuel surcharge and port fees; the language model only explains results and never sets a price.

---

## 5. How the Copilot uses SQL and RAG

1. **Taught answers** are checked first (also for rephrased questions).
2. **SQL layer:** counts, rankings and lists come from the live tables through `copilot_pro.py` INTENTS and the read-only NL→SQL in `qa.py`. Every number comes from SQL.
3. **Agents:** questions such as "what needs attention?" or "why is this quote on hold?" use the latest agent run.
4. **RAG:** knowledge questions (Incoterms, HS codes, demurrage, documents, trade finance) are answered from BM25 + vector search with a cross-encoder reranker, with numbered sources.
5. If nothing relevant is found the Copilot says **"I do not know"** and logs the question to the backlog for Teach Copilot.

---

## 6. Milestone 3 work

### 6.1 Knowledge base (RAG)
- 265 seed websites, `MAX_CRAWL_DEPTH = 3`, `MAX_PDFS_PER_SEED = 100`, chunks of 400 words with 50 overlap.
- Topics: Incoterms, customs and HS codes, shipping documents, containers and cargo, freight rates and surcharges, demurrage and detention, ports and congestion, weather and route risk, marine insurance, trade finance.
- Two fixes to the builder: Google Drive is mounted **before** the crawl, and a **bridge cell** feeds the crawled chunks into `synthetic_kb/all_chunks.jsonl`, the file the app loads. A counter cell at the end prints files, documents and chunks.
- **Sources and chunks collected** (copy these from the last cell of the RAG notebook and add the screenshot):

| Measure | Value |
|---|---|
| Seed websites crawled | 265 |
| PDFs / files saved to `MyDrive/FreightQuote_AI/rag_pdfs/` | **[fill in]** (target 150+) |
| Distinct documents in the index | **[fill in]** |
| Chunks loaded by the app (Admin → System check → Knowledge base) | **[fill in]** |


### 6.2 Copilot fine-tuning
- 22 new SQL intents in `copilot_pro.py`: carriers by on-time and risk, shipments by status/carrier/destination/origin, congested ports, dwell time, quotes by status, largest and thin-margin quotes, margin by tier, restricted goods, highest duty, customs holds, tonnes in transit, slowest and nearly arrived vessels, calm ports, open alerts.
- About 40 extra freight synonyms in `qa.py` (bunker, tonnage, port stay, swell, tariff and so on).
- **Answer rate (Admin → Copilot insights):** before **[fill in] %**, after **[fill in] %**.

### 6.3 Simulators (`sim_fq.py`)
All five simulators read their starting values from the database, never write to it, and share these rules:
- a **seed** field (same seed, same result),
- an **Assumptions** box on the page,
- a **P10 / P50 / P90** range from 1,000 Monte Carlo runs,
- a **"Change vs your previous run"** card for live before/after comparison.

| Simulator | What was added |
|---|---|
| Route | Fuel price change; canal and strait closures (Suez, Panama, Malacca, Hormuz, Bosphorus); route comparison on the four worst lanes showing normal routes and the scenario route (closed routes cannot be picked as cheapest or fastest); cost and extra-days ranges |
| Quote | Margin shift, currency and fuel changes; win probability by price; price band (profit within 5% of best, searched from −10% to +25%); P10–P90 profit band |
| Carrier | On-time reliability drop and capacity shortage; carrier ranking under the disruption; ranges for unplaced shipments and extra cost |
| Weather | Storm intensity (wind with Beaufort label) and duration; delay days and demurrage cost; cost-versus-duration chart |
| Customs | Inspection rate and extra days; missing-document rate and delay; holding cost per day; cost by cause |

Other fixes: the storm simulator raised an error when vessel data was present, and the Monte Carlo panels showed rupees instead of dollars.

### 6.4 Milestone 2 work in the final app
JWT login, Profile page (with profile picture), role-based access with 403 refusals, Admin dashboard (role changes, unlock, deactivate) and IST timestamps run inside the final app.

---

## 7. Honest limits

- Vessel positions, shipments, rates and the price model's training data are simulated; weather and waves are real (Open-Meteo). Connect your own AIS and rate feeds for production.
- The agents are specialist models coordinated by planner graphs, not free-form LLM agents; this is a safety choice.
- Email, Slack and SMS alerts need your own credentials.
- The Copilot cannot cover every phrasing; unanswered questions are logged so they can be taught.

---

## 8. Screenshots (`screenshots/`)

1. RAG notebook: source count and chunk count (last cell)
2. Teach Copilot showing 50 / 50
3. Copilot answer with sources `[1] [2]`
4. Each agent simulator, before and after

| Agent | Before | After |
|---|---|---|
| Route | <img src="screenshots/sim_route_before.png" width="420"> | <img src="screenshots/sim_route_after.png" width="420"> |
| Quote | <img src="screenshots/sim_quote_before.png" width="420"> | <img src="screenshots/sim_quote_after.png" width="420"> |
| Carrier | <img src="screenshots/sim_carrier_before.png" width="420"> | <img src="screenshots/sim_carrier_after.png" width="420"> |
| Weather | <img src="screenshots/sim_weather_before.png" width="420"> | <img src="screenshots/sim_weather_after.png" width="420"> |
| Customs | <img src="screenshots/sim_customs_before.png" width="420"> | <img src="screenshots/sim_customs_after.png" width="420"> |

5. Admin → Copilot insights, before and after teaching
