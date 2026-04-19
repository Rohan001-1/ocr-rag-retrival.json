# OCR RAG System — Retrieval Workflow

An n8n workflow that receives a user question via webhook, searches a Pinecone vector database for relevant content, and returns an AI-generated answer using GPT-4.1 Mini. Supports multi-turn conversation memory per user session.

---

## How It Works

```
Webhook (POST) → AI Agent → Pinecone Vector Store (search) → GPT-4.1 Mini → Respond to Webhook
```

The AI Agent automatically searches the Pinecone index before answering every question. It uses session-based memory so each user has their own conversation history.

---

## Requirements

### Credentials

You will need to set up the following credentials in your n8n instance:

| Node | Credential Type |
|------|----------------|
| OpenAI Chat Model | OpenAI API |
| Embeddings Mistral Cloud | Mistral Cloud API |
| Pinecone Vector Store | Pinecone API |

### Pinecone Index Setup

- Use the **same index** that was used in the Ingestion workflow
- The index dimension must be **1024** (Mistral embedding size)
- After importing, open the Pinecone Vector Store node and select your index

---

## Webhook Configuration

- **Method:** POST
- **CORS:** Set Allowed Origins to `*` in the Options section (to allow browser requests)
- **Request body must include:**

```json
{
  "message": "your question here",
  "sessionId": "unique-user-id"
}
```

### sessionId — Why It Is Required

Each user must send a unique `sessionId` with every request. This allows the workflow to maintain separate conversation memory for each user.

**Recommended approach — generate in the frontend:**

```javascript
let sessionId = localStorage.getItem('sessionId');
if (!sessionId) {
  sessionId = crypto.randomUUID();
  localStorage.setItem('sessionId', sessionId);
}

fetch('YOUR_WEBHOOK_URL', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    message: userMessage,
    sessionId: sessionId
  })
});
```

---

## AI Agent Configuration

### Model
- **Node:** OpenAI Chat Model
- **Model used:** `gpt-4.1-mini`
- You can switch to any other OpenAI model from the node settings

### Memory
- **Node:** Simple Memory (Window Buffer)
- **Session key:** `{{ $json.body.sessionId }}`
- Memory is scoped per sessionId — different users will not share conversation history

### Retrieval
- **Node:** Pinecone Vector Store (retrieve-as-tool mode)
- **Top K:** 15 — the agent fetches the 15 most relevant chunks per search
- The agent will search automatically before answering any question
- If the first search returns no result, the agent will retry with different keywords

---

## System Prompt Rules (already configured)

The AI Agent is instructed to:

- Always search the document before answering
- Never use markdown, asterisks, hashtags, or bullet symbols
- Write in clean plain text only
- Use numbered lists when listing items
- Never start answers with phrases like "According to the document"
- If information is truly not found after multiple searches, respond with: *"This information is not present in the uploaded documents."*

You can edit these rules inside the **AI Agent node → System Message** field.

---

## Important Notes

- **Activate the workflow** before testing from a browser — test mode webhook URL is different from production URL
- This workflow only handles **retrieval and answering**. To upload and store documents, use the companion Ingestion workflow
- Both workflows must use the **same Pinecone index** — otherwise the retrieval will return no results

---

## Related Workflows

- **OCR RAG System — Ingestion:** Handles PDF/image upload, OCR text extraction, and storing data into Pinecone.
