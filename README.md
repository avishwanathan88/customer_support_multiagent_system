# NovaDesk - A Modern AI Helpdesk for Customer Support

*Kaggle Agents Intensive Capstone Project - Enterprise Agents Track*

## Problem Statement

NovaDesk addresses a common challenge in customer support workflows. When a customer submits a free-text message—whether it's a billing complaint or a login problem—human agents must:

- Read and carefully analyze the message
- Classify the type of issue
- Search internal FAQs for relevant information
- Check metrics to identify potential wider incidents
- Draft a clear customer reply
- Write a short internal note for team records

This sequence is highly repetitive and involves constant context switching between tools and tabs. It's also prone to inconsistency—two different agents might classify the same ticket differently or reference conflicting snippets from policy documents.

## The Solution

NovaDesk centralizes this entire workflow into one AI helpdesk system that handles classification, retrieval, metrics analysis, and response formatting while keeping the process transparent and inspectable.

## Why Agents?

The support workflow naturally divides into specialized roles, making it ideal for a multi-agent architecture:

- **Triage & Routing** – Categorizing tickets by type
- **Knowledge Base Search** – Reading FAQs and policy content
- **Metrics Analysis** – Evaluating numerical data and trends
- **RAG Retrieval** – Accessing long-form reference documents
- **Response Writing** – Composing customer messages and internal summaries

Rather than forcing a single large prompt to handle everything at once, NovaDesk uses the Google Agent Development Kit to express these roles as separate agents. Each agent has:

- Its own concise instructions
- Dedicated tools for its specific function
- A clear responsibility within the workflow

This mirrors how real support teams are organized and keeps prompts small, tool usage explicit, and reasoning easier to debug.

## Architecture Overview

```
customer -> orchestrator ---------------------> Triage Agent
      |              |             <------------------------------- |
      |                                                  
      |              |   -----> Internal knowledge agent, business metrics agent
      |                                <-------------------------------|
      |           
      |              |   [CONDITIONAL]-----> RAG Agent 
      |                                <---------------------------|
      |                     
      |              | ----------------------------> Response Formatter Agent
      |                                <------------------------------------|
      |              |                                                                   
      |<-------------|
```

NovaDesk implements a complete multi-agent helpdesk system with the following structure:

### Core Components

#### Orchestrator Agent
- Root agent using Gemini 2.5 Flash Lite model
- Determines if messages are casual conversation or support requests
- Manages the workflow sequence and coordinates all other agents

#### Triage Agent
- Classifies tickets into categories: billing, login, performance, or feedback
- Stores classification in `session.state["query_category"]`

#### Parallel Agent (Knowledge + Metrics)
Runs two agents simultaneously for efficiency:

- **Knowledge Agent** – Uses `search_faq_tool` to query a pandas DataFrame (`faqs_df`) containing internal Q&A grouped by category
- **Metrics Agent** – Uses `business_metrics_tool` to analyze timestamped counts from `metrics_df` and identify recent trends

#### RAG Agent
- Deployed as a separate A2A endpoint on port 8001
- Uses `query_rag_corpus` tool to call Vertex AI RAG retrieval
- Returns structured chunks from the configured corpus
- Invoked when FAQ and metrics context are insufficient

#### Response Formatter Agent
- Reads all context from session state (FAQ, metrics, RAG results)
- Produces final Markdown output with two sections:
  - **Reply** – Customer-facing response
  - **Internal Note** – Summary for support staff

### Session Management
- `InMemorySessionService` for state management
- `EventsCompactionConfig` with interval of 3 and overlap of 1
- Older conversation events are summarized while recent turns remain explicit

## Demo

The notebook demonstrates NovaDesk through two interfaces:

### 1. Debug Mode
Uses `InMemoryRunner` with `LoggingPlugin`

Provides step-by-step trace showing:
- When each agent is invoked
- What tools are called and in what order
- Output from each stage of the workflow
- Whether RAG retrieval was triggered

### 2. Interactive Console

- `CustomerSupportSession` class provides a simple chat interface
- Users type support questions and receive formatted responses
- Shows executed step count for each query
- Tracks conversation turns and allows clean exit with "quit", "exit", or "bye"

## The Build

### Technologies Used

- **Google Agent Development Kit (ADK)** – Multi-agent orchestration
- **Gemini 2.5 Flash Lite** – LLM model
- **Vertex AI RAG** – Retrieval-augmented generation
- **Pandas** – In-memory data storage for FAQs and metrics
- **Python** – Core implementation language

### Implementation Details

#### Data Layer
- `faqs_df` DataFrame: Internal Q&A grouped by category
- `metrics_df` DataFrame: Timestamped support metrics

#### Tool Functions
- `query_rag_corpus` – Vertex AI RAG retrieval
- `search_faq_tool` – FAQ database queries
- `business_metrics_tool` – Metrics analysis
- All tools return structured dictionaries with status fields (success/failed)

#### Infrastructure
- Credentials loaded from Kaggle environment (`GOOGLE_API_KEY`, `GCP_SERVICE_ACCOUNT_KEY`)
- RAG agent deployed as HTTP server using ADK utilities
- `RemoteA2aAgent` consumes the RAG endpoint
- `LoggingPlugin` for observability
- Session compaction for conversation history management

## Future Improvements

### Immediate Priority: Stabilize Orchestration Logic

The current prototype has some unpredictability in multi-turn conversations:
- Some queries might not produce final responses in interactive mode
- The agent might not always follow the intended sequence: triage → FAQ/metrics → optional RAG → response formatter due to the LLM making the code flow decisions

**Planned fixes:**
- Debug and tighten orchestration instructions
- Enforce that all non-greeting messages go through the full workflow
- Add structured checks to make tool calling patterns more robust

### Long-Term Enhancements

#### System Integration
- Connect FAQ and metrics tools to real knowledge bases and analytics systems
- Maintain existing tool signatures for easy migration

#### Enhanced Capabilities
- Expand triage categories with additional fields (product, region, priority)
- Tune RAG retrieval parameters
- Surface more metadata in internal notes

#### User Experience
- Build a minimal web interface for non-technical users
- Create analytics dashboard from `LoggingPlugin` data
- Analyze when FAQ/metrics alone suffice vs. when RAG adds value

#### Continuous Improvement
- Study stored sessions to identify refinement opportunities
- Optimize prompts based on real usage patterns
- Measure impact of architecture changes

## Authors

- **Avishwan** - [Kaggle Profile](https://www.kaggle.com/avishwan)
- **Anusree Mondal Rakhi** - [Kaggle Profile](https://www.kaggle.com/anusreemondal)
