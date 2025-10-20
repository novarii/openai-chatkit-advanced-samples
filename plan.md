## Task Plan

1. Survey product requirements and target workflows to scope the agent’s responsibilities, available tools, and UI surfaces.
2. Prepare the runtime: install ChatKit + Agents SDK dependencies, configure environment variables (e.g., `OPENAI_API_KEY`), and choose storage adapters for threads, items, and attachments.
3. Implement the ChatKit server wrapper that instantiates your Agent(s), registers tools, and streams responses via `Runner.run_streamed` and `stream_agent_response`.
4. Build or adapt tool implementations, widgets, and REST endpoints so the agent can trigger client-side actions, render rich UI components, and synchronize state with the frontend.
5. Validate end-to-end behavior with local runs (backend + frontend), iterating on prompts, guardrails, and persistence before hardening for deployment.


  Integration Report

  - Environment prep: Install ChatKit + Agents SDK via the backend bootstrap in (FastAPI) and ensure OPENAI_API_KEY is exported before running npm run backend so the agent can call OpenAI models.
  - FastAPI entrypoint: The /chatkit route streams ChatKit protocol events by delegating payloads to FactAssistantServer.process, returning StreamingResponse when the SDK yields StreamingResult
  - Server wiring: FactAssistantServer wraps the Agents SDK: it builds an in-memory MemoryStore (backend/app/chat.py:204-214, backend/app/
    memory_store.py:17-148), instantiates an Agent[FactAgentContext] with instructions from constants.py and registered tools, and streams
    results via Runner.run_streamed + stream_agent_response (backend/app/chat.py:208-247), matching the pattern described in ChatKit Python docs
    “Define ChatKit Server Class with Agent SDK Integration”.
  - Thread → agent input: Incoming ThreadItems are converted through ThreadItemConverter if available and fall back to raw user text when needed
    (backend/app/chat.py:252-334), aligning with ChatKit docs on ThreadItemConverter for attachments, tags, and hidden context; extend this when
    enabling files or structured history.
  - Tool delegation: Tools are declared with @function_tool, mutate ctx.context.client_tool_call, and return structured output (backend/app/
    chat.py:90-189); this mirrors the “Trigger Client-Side Tools from Agent SDK” guidance so ChatKit widgets and client tools fire once the SDK
    stops at tools.
  - Widgets & client state: get_weather renders a ChatKit widget and streams it with ctx.context.stream_widget, while save_fact emits
    hidden context markers via ThreadItemDoneEvent to keep thread state in sync (backend/app/chat.py:75-188); the widget components live in
    sample_widget.py and use ChatKit’s widget primitives.
  - State management: MemoryStore fulfills the ChatKit Store contract for threads/items (backend/app/memory_store.py:17-148); replace with a
    persistent store (Postgres, S3, etc.) following the ChatKit docs once deploying beyond demos.
  - Prompt & guardrails: System behavior is centralized in INSTRUCTIONS and the target model string in MODEL (backend/app/constants.py:7-36),
    making it easy to add guardrails or switch models, echoing Agents SDK recommendations on top-level agent configuration.
  - Extending the demo: Add new tools by defining further @function_tools, append them to the tools list before agent initialization, expose
    any REST endpoints the frontend needs, and use docs’ examples (e.g., StopAtTools, previous_response_id preservation) when layering richer
    workflows or persistent memory.
  - Client/front-end pairing: The root README outlines how the Vite proxy forwards /chatkit calls to FastAPI (README.md:3-89); ensure your
    domain key and allowlist are configured when deploying secure hosts.