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

The Final Priority Score (0-200) is calculated as follows:

### 1. AI Semantic Score (0-100)
Calculated using Cosine Similarity ($sim$) against three anchor types: **Risk**, **Polite**, and **Ignore**.

$$
sim(u, v) = \frac{u \cdot v}{\|u\| \|v\|}
$$

**Logic:**
*   If `ignore_sim > risk_sim` $\rightarrow$ **Score: 0**
*   If `polite_sim > risk_sim * 1.1` $\rightarrow$ **Score: 0**
*   If `risk_sim < 0.42` (Similarity Threshold) $\rightarrow$ **Score: 0**
*   **Default**: $50 + (risk\_sim - 0.42) \times 100$
*   *Explicit Urgency Override*: If specific keywords are found $\rightarrow$ **Score: 85**

### 2. Business Boosters (0-100)
*   **Financial Tier**: $\log_{10}(MRR + 1) \times 5 \times TierMultiplier$ (Max ~30 pts)
*   **Recurrence**: +15 points if the client has >5 past tickets.
*   **Stagnation**: `min(days_open, 20)` points.

**Final Score Formula**:
$$
Score = AI\_Score + Financial + Recurrence + Time
$$

*(If AI\_Score is 0, the Final Score is forced to 0 to avoid false positives)*

**Result**: Precision@30 of **96.67%**.

---

## Cost Analysis (Scale: 100k Tickets)

How much does it cost to process **100,000 tickets** using this architecture?

**Assumptions**:
*   Average ticket length: **300 tokens**.
*   Model: OpenAI `text-embedding-3-small`.
*   Price: **$0.02 / 1M tokens**.

### The Math:
$$
100,000 \text{ tickets} \times 300 \text{ tokens} = 30,000,000 \text{ tokens}
$$
$$
30M \text{ tokens} \times \$0.02 = \mathbf{\$0.60}
$$

### Comparison vs. LLM Approach

| Approach | Model | Cost per 1M Tokens | Monthly Cost (100k tickets) |
| :--- | :--- | :--- | :--- |
| **Our Solution (Embeddings)** | **text-embedding-3-small** | **$0.02** | **$0.60** |
| LLM (GPT-4o-mini) | GPT-4o-mini | $0.15 (In) / $0.60 (Out) | ~$10.50 |
| LLM (GPT-4o) | GPT-4o | $5.00 (In) / $15.00 (Out) | ~$350.00 |

**Conclusion**: This solution is **94% cheaper** than the cheapest LLM and **99.8% cheaper** than GPT-4o, while being faster and more deterministic.

---