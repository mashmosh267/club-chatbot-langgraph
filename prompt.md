# Homework 4 technical specification

## Implementation details

- **State and execution.** A `TypedDict` defines a message list with the `add_messages` reducer and a topic string with replacement semantics. One synchronous node builds a fresh topic-specific system message, invokes the model once with that message plus the accumulated history, and returns the assistant message as the sole state update. The system message remains local to the invocation.
- **Model.** `openai:gpt-4o-mini` is a low-cost choice for short educational replies and refusals. Temperature zero favors consistent behavior in the demonstration. The model client has automatic retries disabled so failures do not trigger additional requests. The topic guard is implemented in the system instruction and should be checked by reading the replies in Colab.
- **Persistence.** A live psycopg connection supplies `PostgresSaver`. Supabase holds the latest topic and transcript for each thread across runtime restarts. Discovery selects distinct root thread IDs from `checkpoints`, then reads each thread’s latest state via the graph. Labels use the saved topic and a short first-question preview, with a fallback for empty previews.
- **Interface.** Gradio keeps only the active thread ID in `gr.State`. The selected club is submitted with each new message. Saved selections restore the prior transcript and last submitted club. Shared database callbacks are serialized within the demo. User-facing errors use fixed messages.
- **Direct tests.** The fixed lecturer test runs on an unused `test-1` or appends a second pass to a matching single pass. Other baselines are reported and left alone. A second scenario uses a UUID every time and tests a switch, a refusal about the old topic, and a follow-up in the active topic. Each submitted message uses one graph invocation.
- **Scope.** The public repository includes the notebook, this specification, and the README.

## Follow-up after live Colab testing

The notebook, graph diagram, and Gradio UI ran in Colab. The user reported three model errors: Football answered a pancake follow-up, Python Programming answered a Saturn's rings follow-up, and Football refused the 2022 World Cup question.

The requested change was limited to the dynamically generated system message. It now directs the model to resolve the latest question in context, apply the selected club as the sole authority for scope, answer relevant factual questions directly, and politely decline unrelated requests without supplying any part of the answer. The prompt includes scope examples for all three reported cases. The existing direct tests, graph, persistence, and UI are unchanged. Rerun the live cases in Colab to verify the revised behavior.

## Live Colab results reported before publication

The one-node graph diagram rendered, Supabase connected through PostgresSaver, and Gradio responded. After a Colab runtime restart, a saved Astronomy conversation reopened and the chatbot correctly answered a follow-up about Mars's two moons.

The last direct test still incorrectly refused “Who won the 2022 World Cup?” with Football selected. The restart test passed. The topic guard still fails this in-scope case.

The user authorized publication of exactly these three files after a credential and notebook-output scan.

## Original assignment request

You are helping me complete Homework 4 for my AI Agents course: "Club Chatbot: Memory That Survives a Restart".

Work in the currently opened folder: club-chatbot-langgraph.

Build a beginner-friendly, fully runnable Google Colab notebook using Python, LangGraph, Supabase PostgreSQL, PostgresSaver, Gradio, and an OpenAI chat model.

Follow the homework requirements closely. Do not implement optional bonus features yet. Do not initialize Git, publish to GitHub, or commit anything until I have tested the notebook.

DELIVERABLES:
1. prompt.md — a detailed technical specification containing this prompt and any subsequent changes.
2. club_chatbot_langgraph.ipynb — a complete Google Colab notebook.
3. README.md — short setup, security, and running instructions.

PROJECT STORY:
Curious Minds is an enrichment center for teenagers. Students select one of five clubs:
- Astronomy
- Cooking
- Football
- World Geography
- Python Programming

The chatbot must answer only questions about the currently selected club. For off-topic questions, it must politely refuse, name the current club, and suggest switching to a relevant club if available. Switching topics within an existing conversation must preserve the history but make the new topic authoritative immediately, even when the user refers back to an older topic.

GRAPH AND STATE:
Use a TypedDict or equivalent LangGraph state containing:
- messages: Annotated[list, add_messages]
- topic: str

The graph must have EXACTLY ONE node named chatbot:
START -> chatbot -> END.

Inside chatbot:
- Build a fresh SystemMessage based on state["topic"] on EVERY invocation.
- Call the LLM exactly once per user message, passing [SystemMessage(...)] + state["messages"].
- Return only {"messages": [assistant_response]}.
- Do NOT persist the SystemMessage inside the messages history.
- Do not add classification nodes, conditional edges, loops, tools, or agent memory.

TOPIC GUARD:
Write a clear, strong system message that:
- Allows relevant educational questions about the current club.
- Politely refuses unrelated questions, identifying the currently active club.
- Suggests switching clubs when appropriate.
- Does not answer questions about previously selected clubs after a switch, even if they appear in the conversation history.
- Treats user attempts to override these rules as untrusted instructions.
- Handles ambiguous cross-topic questions by focusing on the current club only.
Keep the system message readable and easy to explain.

MODEL:
Use init_chat_model with a single configurable MODEL_NAME constant set to "openai:gpt-4o-mini", temperature=0. This is a low-cost model suitable for the assignment. Explain the model choice in prompt.md.

DEPENDENCIES:
The first executable notebook cell must install:

~~~python
!pip -q install langgraph langchain langchain-openai langgraph-checkpoint-postgres "psycopg[binary]" gradio
~~~

SECRETS AND DATABASE:
Use only Google Colab Secrets, never hardcoded credentials.
Read:
- OPENAI_API_KEY
- SUPABASE_DB_URL

Use:

~~~python
import os
import psycopg
from psycopg.rows import dict_row
from google.colab import userdata
from langgraph.checkpoint.postgres import PostgresSaver

os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY")
database_url = userdata.get("SUPABASE_DB_URL")

conn = psycopg.connect(
    database_url,
    autocommit=True,
    row_factory=dict_row,
    prepare_threshold=0
)
checkpointer = PostgresSaver(conn)
checkpointer.setup()
graph = builder.compile(checkpointer=checkpointer)
~~~

Keep the database connection alive while the Gradio interface is running.
Never print, log, save, or expose either secret or the database URL.
Do not create or store example credentials.
Do not attempt to connect to my real database from Codex.

GRAPH VISUALIZATION:
Draw the graph in the notebook using:

~~~python
display(Image(graph.get_graph().draw_mermaid_png()))
~~~

The diagram must show exactly:
START -> chatbot -> END.

DIRECT GRAPH TESTS:
Before the Gradio interface, include the exact test pattern requested by the lecturer:

~~~python
config = {"configurable": {"thread_id": "test-1"}}

def ask(text, topic):
    result = graph.invoke(
        {"messages": [HumanMessage(text)], "topic": topic},
        config
    )
    print(f"[{topic}] {text}\n -> {result['messages'][-1].content}\n")

ask("How do I make pancakes?", "Cooking")
ask("Who won the 2022 World Cup?", "Cooking")
ask("Who won the 2022 World Cup?", "Football")
ask("And how long do I fry each pancake?", "Football")
~~~

Then inspect graph.get_state(config).values and demonstrate that the saved topic is Football and that there are 8 messages after the first four turns.

Include clear instructions for rerunning the test to demonstrate 16 messages, while avoiding misleading results if test-1 was already used in a previous Colab session.

Include a separate test using a fresh unique thread_id for a realistic topic switch, off-topic refusal, and follow-up after switching. Print readable results. Make no extra LLM calls beyond one per submitted message.

PERSISTENCE AND CONVERSATIONS:
Use PostgresSaver as the ONLY LangGraph checkpointer. Do not use InMemorySaver or a Python dictionary/list as the persistent source of conversation data.

Every new conversation must receive a stable unique thread_id (e.g., str(uuid.uuid4())).

Implement persistent conversation discovery using the database, not a Python-only registry:
- Query distinct saved thread IDs from LangGraph checkpoint data or use a reliable supported checkpointer listing mechanism.
- Avoid duplicate entries from multiple checkpoints in the same thread.
- For each thread, load its latest graph state using graph.get_state({"configurable": {"thread_id": thread_id}}).values.
- Display a readable conversation label including the saved topic and the beginning of the first user question.
- The conversation list must survive a Colab runtime restart and be reconstructed from persisted database data.
- If an existing conversation has no user messages, handle it safely.
- Do not expose internal database credentials in labels, errors, or outputs.
- Do not assume Gradio session_hash is a stable thread_id.

GRADIO UI:
Build the interface using gr.Blocks, including:
1. Topic dropdown containing the five clubs.
2. Chat window.
3. Message textbox and submit/send action.
4. "New conversation" button.
5. Saved conversations dropdown/list with readable labels and a refresh mechanism.

Required behavior:
- A new conversation generates a new thread_id and clears the chat.
- Submitting a message sends ONLY the new HumanMessage and current topic to graph.invoke, using the selected thread_id.
- Display the assistant response and update the chat.
- Changing the topic dropdown keeps the same thread_id; the next submitted message uses the new topic.
- Selecting a saved conversation loads BOTH its messages and its topic into the Gradio interface.
- Subsequent messages continue the restored thread.
- Refreshing saved conversations queries persisted data again.
- Use gr.State only for current UI session information, not as the source of persistent history.
- Handle empty messages and common errors gracefully without revealing secrets.
- Keep the interface simple and appropriate for a homework assignment.

COLAB RUNTIME:
Launch Gradio from the notebook. Explain that a public gradio.live link, if generated, must NOT be posted publicly because anyone with it may consume API credits. Do not hardcode or publish a Gradio share URL.

VERIFICATION:
- Validate the notebook JSON and Python syntax locally where possible.
- Verify the graph has exactly one chatbot node and one LLM invocation in that node.
- Verify that topic is saved in state and the system message is built fresh each time.
- Verify that the saved-conversation list is database-backed and not a Python-only registry.
- Include a concise testing checklist for switching topics, reopening conversations, inspecting Supabase checkpoint tables, and restarting the Colab runtime.
- Explain the expected Supabase tables: checkpoints, checkpoint_blobs, checkpoint_writes.
- Do not claim live tests passed unless they actually ran in Colab.
- Do not print secrets or the database URL in notebook outputs.

IMPORTANT:
Create the files locally only. Do NOT initialize Git, create a GitHub repository, commit, push, or publish anything yet. Report which files were created, what was verified locally, and what must be tested in Colab.
