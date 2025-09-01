# Application Logic Flow Documentation

# 

## Step-by-Step Interaction and Logic Flow

This detailed breakdown follows a single user message through the entire application stack.

### Step 1: Frontend - User Sends a Message

**User Action:** The user types a message into the text box (`#messageInput` in `index.html`) and clicks the send button (`#sendBtn`).

**JavaScript Trigger:** The `sendMessage()` function in `static/js/script.js` is called.

**UI Update (Immediate):**

- The user's message is instantly added to the chat window by calling `addMessage('user', ...)`. This provides immediate feedback.
- The input field is disabled to prevent multiple submissions.
- The `showTypingIndicator()` function is called, displaying the "..." animation to show the bot is "thinking."

**API Request:** A fetch POST request is sent to the `/chat` endpoint on the Flask server. The request body is a JSON object containing the user's message and the current `session_id` (or null if it's the first message).

### Step 2: Backend - The API Entry Point (app.py)

**Request Received:** The Flask application in `app.py` receives the POST request at the `@app.route("/chat")` endpoint.

**Validation & Parsing:** The `chat()` function validates that the request is JSON, extracts the message and `session_id`, and ensures the message is not empty. If the `session_id` is missing, a new one is generated.

**Delegation to Core Logic:** This is the most critical handoff. The `app.py` file does not contain any business logic. It immediately delegates the processing by calling `route_message(session_id, message)` from `core/router.py`.

### Step 3: Core Logic - The Message Router (core/router.py)

The `route_message` function executes a strict, prioritized sequence of operations.

#### Persistence & Normalization

- The user's original message is saved to the database via `database/repository_v2.py`.
- The message text is normalized (converted to lowercase, etc.) by `nlp/preprocessor.py`.

#### Priority 1: Safety - Risk Detection

- The normalized message is passed to `contains_risk()` in `nlp/risk_detector.py`.
- This function checks the message against a comprehensive list of high-risk keywords and phrases from `config/risk_phrases.json`.
- **If risk is detected:** The router immediately stops all other processing. It calls `get_crisis_resources()` to get the pre-formatted safety message and returns this as the final reply. The conversation flow is halted for safety.

#### Session & State Retrieval

- If no risk is detected, the router calls `get_session(session_id)` from `core/session.py`.
- This retrieves the user's current session, including their `ChatBotFSM` object, which holds their current state (e.g., 'support_people'). If the session is new, it creates one with the initial state of 'welcome'.

#### Logic Branching (FSM vs. LLM)

The router inspects the current FSM state to decide which logic path to take.

### Path A: Structured FSM Conversation (The "Happy Path")

This path is taken if the state is anything before 'summary' (e.g., welcome, support_people, strengths, etc.).

#### NLP Analysis

It first performs two key NLP tasks:

- `classify_intent()` (nlp/intent_distilbert.py): Determines the user's intent (e.g., 'support_people', 'strengths', 'negation').
- `analyze_sentiment()` (nlp/sentiment.py): Determines the emotional tone (e.g., 'positive', 'negative').

#### State-Specific Handling

The router calls the specific function for the current state (e.g., `_handle_support_people_state()`).

#### Business Logic

Inside the state handler, it decides what to do based on the classified intent:

**If Intent Matches Step:**

- The user's response is considered valid.
- The response is saved to the FSM object (`fsm.save_response(...)`).
- The FSM is transitioned to the next step (`fsm.next_step()`).
- The new FSM state is saved to the database.
- A reply is constructed using `nlp/response_selector.py` which includes an acknowledgment for the current step and the prompt for the next step.

**If Intent Does NOT Match:**

- The progressive fallback system is triggered.
- An attempt counter for that step is incremented.
- Based on the attempt count, `nlp/response_selector.py` provides a clarifying prompt or an option to move on.
- The FSM state does not advance.

### Path B: Open-Ended LLM Conversation

This path is taken if the state is 'summary' or 'llm_conversation', indicating the structured 4-step process is complete.

- The router calls `_handle_llm_conversation()`, which uses the `llm/handoff_manager.py`.

#### Context Building

The `LLMHandoffManager` retrieves the user's FSM responses (support, strengths, etc.) and the last few turns of conversation from the database to build a rich, personalized context.

#### LLM Call

It constructs a system prompt and sends the context + user message to the LLM via `llm/client.py`.

#### Guardrails

The raw response from the LLM is passed through `llm/guardrails.py` to filter out PII, inappropriate content, and ensure it meets length and tone requirements.

The final, sanitized LLM response is returned.

#### Final Reply

The router returns the final composed reply string (whether from the risk detector, FSM handler, or LLM handler) and debug information back to `app.py`.

### Step 4: Backend - The API Exit Point (app.py)

**Response Formatting:** The `chat()` function in `app.py` receives the reply from the router. It gets the FSM's latest state.

**JSON Creation:** It constructs the final JSON response payload, including the reply, `session_id`, current state, and any debug info.

**Response Sent:** The Flask server sends this JSON object back to the user's browser with a 200 OK status.

### Step 5: Frontend - Receiving and Displaying the Response

**API Response Handled:** The `.then(data => ...)` block in the `sendMessage()` function in `static/js/script.js` executes.

**UI Update:**

- The `sessionId` variable is updated for the next request.
- `hideTypingIndicator()` is called to remove the "..." animation.
- `addMessage('bot', data.reply)` is called, displaying the bot's response in a new chat bubble.
- `updateDebugPanel()` is called to show the latest state and debug info.
- The input field is re-enabled, and the cursor is focused, ready for the user's next message.

This completes one full cycle of the application's logic, demonstrating a safety-first, structured, and extensible flow.
