# Music Store Customer Support (LangGraph multi-agent)

A customer-support assistant for a digital music store, built on the Chinook sample database. A customer first proves who they are (customer ID, email or phone). The system then loads their saved music preferences and passes the question to a supervisor agent. The supervisor routes it to a music-catalog agent, an invoice agent, or both, and writes one combined answer. At the end of the run, the conversation is used to update the customer's saved preferences.

## Flow

```
START -> verify_info -(verified)-> load_memory -> supervisor -> create_memory -> END
              |                                    |-> music_catalog
        (not verified)                             |-> invoice_info
              v
         human_input -> verify_info      (after 3 failed attempts: END)
```

| Node | What it does |
|---|---|
| `verify_info` | Extracts credentials from the latest message with structured LLM output, then looks the customer up in the database. Phones are compared digits-only and emails case-insensitively. |
| `human_input` | Calls `interrupt()` to pause the graph and ask for credentials. The run resumes with `Command(resume=...)` on the same `thread_id`. |
| `load_memory` | Copies the customer's saved preferences from the store into `State["loaded_memory"]`. |
| `supervisor` | Routes to one or both sub-agents and writes one combined answer from their results. |
| `create_memory` | Extracts the artists and genres the customer mentioned, merges them with the saved ones (no duplicates) and saves them. |

## Components

- **`Settings` / `get_llm()`**: configuration read from `.env`, and one place that builds the chat model.
- **`ChinookDatabase`**: downloads the Chinook SQL script into an in-memory SQLite database (`StaticPool`, `check_same_thread=False`).
- **`State`**: `messages`, `customer_id`, `loaded_memory`, `remaining_steps`, `auth_attempts`.
- **`MusicCatalogTools`**: albums by artist, tracks by artist (plus similar artists in the same genre), songs by genre, and check whether a song exists.
- **`InvoiceTools`**: invoices by date, invoice lines by unit price, and the support employee for an invoice.
- **`BaseAgent`, `MusicCatalogAgent`, `InvoiceAgent`**: ReAct agents. Each has only its own tools. The music agent's prompt includes the customer's saved preferences. The invoice agent's prompt includes the verified `customer_id`.
- **`SupervisorAgent`**: built with `langgraph-supervisor`.
- **`CustomerVerifier`, `HumanInputNode`**: verification and the human-in-the-loop pause.
- **`MemoryManager`, `LoadMemoryNode`, `CreateMemoryNode`**: long-term memory, stored per customer under the namespace `("memory_profile", customer_id)`.
- **`SupportGraphBuilder`**: wires everything into a `StateGraph`, compiled with a checkpointer and a store.
- **`SupportApp`**: a small wrapper with `chat()` and `resume()` that handles interrupts.

## Security notes

- All SQL passes user values as bound parameters (`:name`), never f-strings, so input like `x' OR '1'='1` cannot change the query.
- Every invoice query filters on the verified `customer_id`, and the invoice agent is told to refuse requests about other accounts.
- The `give_up` route and the `auth_attempts` counter end the verification loop after 3 failed attempts.

## Setup

Requires Python 3.10+ (developed on 3.14) and an OpenAI API key.

1. Create and activate a virtual environment, then install the dependencies:

   ```bash
   python -m venv .venv
   .venv\Scripts\activate        # Windows
   source .venv/bin/activate     # macOS / Linux
   pip install -r requirements.txt
   ```

2. Create a `.env` file next to the notebook:

   ```
   OPENAI_API_KEY=your-key-here
   MODEL_NAME=your-chat-model-name
   ```

   Never commit `.env` or include it in a submission zip.

3. Open `Music_Store_Support.ipynb` and run the cells top to bottom. The first run downloads the Chinook script, so it needs internet access.

## Usage

```python
app = SupportApp()

# Non-interactive: returns "[INTERRUPT] <prompt>" when credentials are needed
app.chat("What was my most recent invoice?", thread_id="demo", interactive=False)
app.resume("my email is luisg@embraer.com.br", thread_id="demo")

# Interactive: reads answers from input()
app.chat("Recommend some Rolling Stones albums", thread_id="demo2")
```

## Tests

Section 10 of the notebook runs two scenarios:

1. **Test case 1**: phone number plus an invoice question and Rolling Stones albums in one message. Then a preference-based follow-up on the same thread, and again on a new thread to show that long-term memory carries over.
2. **Test case 2**: no credentials, so the graph pauses. A wrong phone number is rejected, then a valid email resumes the run. A separate thread shows the loop guard ending the run after 3 bad answers.

## Limitations

- Memory and checkpoints are in RAM and are lost on restart.
- The database is rebuilt from a download on every start.
- "Similar artists" just means artists with tracks in the same genre.
- Preference extraction depends on the LLM.
- Verification trusts the credentials typed in chat, with no password.
- `create_react_agent` is deprecated in LangGraph 1.x in favor of `langchain.agents.create_agent`. It still works.
- A harmless `wrote to unknown channel remaining_steps` message may appear in the output.

## Stack

Python, LangChain, LangGraph, `langgraph-supervisor`, an OpenAI chat model, SQLAlchemy, SQLite, Jupyter.
