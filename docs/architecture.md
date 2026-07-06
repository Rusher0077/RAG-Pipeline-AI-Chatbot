# Architecture

## Overview

The workflow is made up of two independent flows that share one Pinecone index. One handles ingestion, the other handles the actual chat. They don't depend on each other, ingestion can run with no chat session active, and the chat agent doesn't need to know anything about how the index was built.

**Ingestion pipeline**
1. Google Drive Trigger (fileUpdated) and Google Drive Trigger1 (fileCreated) both watch the same folder
2. Download Document
3. Parse Document Text
4. Recursive Character Text Splitter
5. Embeddings (Ingestion)
6. Pinecone Vector Store

**Query pipeline**
1. When chat message received
2. AI Agent, calling Query Knowledge Base as a tool when needed
3. Google Gemini Chat Model, using Conversation Memory for context
4. Response returned to the user

*(Numbers represent order of execution)*

## Ingestion Pipeline, Node By Node

**Google Drive Trigger and Google Drive Trigger1**

Both nodes watch the same folder, but for different events, one for `fileUpdated`, one for `fileCreated`. This wasn't the original design. At first there was only one trigger watching `fileUpdated`, and it worked fine during early testing, but later it stopped firing reliably on new uploads. After some digging (and finding other people on the n8n community forum running into the same thing), it turned out this is a known limitation with how n8n's Drive trigger detects changes, it doesn't always fire consistently for every event type. Adding a second trigger node for the other event type fixed it. Both feed into the same downstream flow.

**Download Document**

Pulls the binary content of whatever file triggered the flow.

**Parse Document Text**

Takes the binary file and extracts its text. Both source documents are `.docx`, so this node handles the parsing before anything gets chunked.

**Recursive Character Text Splitter**

Splits the parsed text into chunks of 1000 characters with no overlap. The documents are short and reasonably well structured (headers, distinct sections), so overlap wasn't needed to keep context intact across chunk boundaries.

**Embeddings (Ingestion)**

Runs each chunk through Google Gemini's embedding model, producing the vectors that get stored.

**Pinecone Vector Store**

Upserts the embedded chunks into a Pinecone index. Both documents share a single namespace, so the agent can pull from either one without needing to know in advance which document an answer might come from.

## Query Pipeline, Node By Node

**When chat message received**

The entry point for a user message.

**AI Agent**

This is where the actual reasoning happens. It's connected to a chat model, a memory node, and a retrieval tool, and it decides on its own when it needs to call the tool versus when it can answer directly. It runs under a system prompt that sets the persona (Nova, an assistant for a fictional company called NovaBridge) and a fairly strict set of rules, most importantly, that it has to check the knowledge base before answering anything, and it's not allowed to fall back on its own general knowledge if the documents don't cover something.

**Conversation Memory**

Keeps track of the conversation so far, so follow up questions don't need to repeat context the user already gave. This was added after the first version of the workflow, which had no memory at all, meaning every message was answered in isolation with no awareness of anything said before it.

**Query Knowledge Base**

This is the Pinecone retrieval tool, set up as something the agent can call rather than context that's automatically fed in. The agent decides when to use it, which is what the system prompt's rule about always checking the knowledge base actually enforces in practice, without that rule, nothing would force the agent to call this tool at all.

**Embeddings (Query)**

Embeds the user's question so it can be compared against the stored document vectors. This is a separate step from the embeddings used during ingestion, it runs fresh for every question.

**Google Gemini Chat Model**

Generates the final response, using only what was retrieved from Pinecone. Originally this was set up with an OpenRouter free model, which worked but was noticeably slower. It's since been switched to Gemini, which responds much faster.

## What's Actually Been Verified

- A short FAQ and policy document was indexed and queried, the agent answered correctly using the retrieved content
- A second, longer document was added to the watched folder in a separate session (not immediately after the first test, a real gap in time), and the trigger picked it up on its own
- A question that could only be answered from that second document was answered correctly, with nothing pulled in from the first document by mistake

This has been tested end to end with two documents. It hasn't been tested yet with a larger knowledge base, multiple people using it at once, or documents that actually contradict each other, though the system prompt does include instructions for how to handle that last case if it comes up.

## Things Worth Knowing If You Look Closely At The Logs

Asking a single question that touches more than one topic can trigger the agent to call the retrieval tool more than once before it answers. For example, a broad question like "what are the office hours" ended up pulling both employee working hours and customer support hours in separate retrieval passes before the agent combined them into one answer. This isn't a bug, it's the agent being thorough per the system prompt's instruction to address every part of a multi part question, but it does mean token usage and response time can climb faster than you'd expect for what looks like a simple question. Increasing the number of results returned per retrieval call can reduce how often this happens.

## Known Limitations

- No error handling if a file fails to download, gets parsed incorrectly, or an embedding call fails
- Only tested with two documents, so retrieval quality at real scale is unverified
- No deduplication safeguard, re-running the ingestion flow manually on a file that's already indexed will re-embed and re-upsert it rather than skipping it
- Runs on a local, self hosted n8n instance, not deployed anywhere persistent or publicly reachable