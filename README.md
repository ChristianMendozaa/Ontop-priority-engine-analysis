# Ontop Priority Engine - Analysis & Methodology

## Project Repositories

This analysis powers the full stack application:

*   ** Frontend Dashboard**: [https://github.com/ChristianMendozaa/OnTop-priotiy-engine-frontend](https://github.com/ChristianMendozaa/OnTop-priotiy-engine-frontend)
*   ** Backend API**: [https://github.com/ChristianMendozaa/OnTop-Priority-Engine-Server](https://github.com/ChristianMendozaa/OnTop-Priority-Engine-Server)

## Live Demo
Check out the live application here: [**OnTop Priority Engine Dashboard**](https://on-top-priotiy-engine-frontend.vercel.app/)

## The core insight
During our exploratory data analysis (EDA), we discovered a critical issue: **Data Labeling Corruption**.
In ~7.5% of conversations, the `sender_type` metadata (Customer vs Agent) was incorrect. Agent responses were labeled as Customer messages, and vice versa.

### Implications
*   **Rule-based systems fail**: We cannot rely on "Who sent the last message?" to detect ghosting.
*   **Sequential Analysis fails**: Traditional LSTM/RNN approaches would be confused by the broken sequence flow.

---

## The Solution: Semantic Embedding Engine

We moved away from metadata-dependency and built a **purely semantic** priority engine. 

### how it works (Embeddings > LLMs)
While Large Language Models (LLMs) are powerful, they are slow and expensive for real-time scoring of thousands of tickets. We used LLMs **only once** during the research phase to identify patterns, but the production engine relies 100% on **Vector Embeddings**.

#### 1. Semantic Anchors (The Reference Points)
We used an LLM to cluster thousands of historical messages and distill them into "Golden Anchors"—synthetic messages that perfectly represent specific urgency states:
*   **URGENT_RISK**: *"I have been waiting for weeks", "Where is my money?", "This is unacceptable"*
*   **POLITE_WAITING**: *"Just checking in", "Thanks for the update", "Let me know when it's done"*
*   **IGNORE**: *"Ticket resolved", "I will close this now", "Happy to help"*

#### 2. Vector Scoring (The Engine)
In production, we simply:
1.  Convert the incoming ticket's latest message into a vector (using `text-embedding-3-small`).
2.  Measure the **Cosine Similarity** between this message and our Golden Anchors.
3.  If the message is mathematically closer to **URGENT_RISK** than **IGNORE**, it gets a high score.

**Why this is better:**
*   **Label Agnostic**: It detects urgency even if the metadata says "Agent".
*   **Ultra Fast**: Cosine similarity is a simple matrix multiplication, milliseconds per ticket.
*   **Deterministic**: No hallucinations. The anchors are fixed.

---

## The Algorithm

The Final Priority Score (0-200) is a weighted sum of:

1.  **AI Semantic Score (0-100)**: Based on embedding distance to Risk Anchors.
2.  **Financial Tier (MRR)**: Logarithmic boost for high-value clients.
3.  **Tempo & Recurrence**: Penalties for stagnation (days open) and repeated issues (historical frequency).

**Result**: Precision@30 of **96.67%**.

---