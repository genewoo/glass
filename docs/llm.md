# LLM Integration

Glass orchestrates multiple services to talk with large language models (LLMs) for live conversation summaries and interactive Q&A. This unified design document consolidates the runtime behavior and prompts used across the application.

## Provider Abstraction

LLM access is routed through provider modules so higher-level services remain provider-agnostic.

### Factory overview

- The provider factory in `src/features/common/ai/factory.js` normalizes provider identifiers, sanitizes model names, and instantiates the correct handler for both synchronous and streaming calls.
- `createLLM(provider, opts)` and `createStreamingLLM(provider, opts)` fetch the handler from the `PROVIDERS` map, rewrite `openai-glass` to the OpenAI handler, and then call the provider-specific constructor.
- Additional providers (Anthropic, Gemini, Ollama, etc.) expose the same surface area, so services only supply provider id, model, API key, and optional Portkey parameters.

### OpenAI handler

- The OpenAI provider builds chat-completion clients. Standard requests wrap the `openai` SDK or a Portkey proxy and return both the generated text and the raw response.
- Streaming requests always perform an HTTP POST with `stream: true` so callers can process Server Sent Events (SSE) tokens incrementally.

### Prompt templating utilities

- `getSystemPrompt` injects live conversation transcripts or screen context into the template before messages are sent, ensuring every request carries the latest meeting state.

## Summary Workflow

`SummaryService` (`src/features/listen/summary/summaryService.js`) keeps a rolling view of the meeting transcript and periodically asks the model for a structured recap.

### Conversation buffering

- Each transcript turn is recorded as `"speaker: text"` via `addConversationTurn`, which trims whitespace and appends the value to `conversationHistory`.
- `triggerAnalysisIfNeeded` runs whenever a turn is added and dispatches a summary request each time the history length is a multiple of five (5, 10, 15, …) so recaps update at predictable intervals.
- `formatConversationForPrompt` slices the most recent 30 turns so prompts stay within model limits. Older dialogue is dropped, while previous analysis is compressed into a small `contextualPrompt` containing the last topic, up to three key points, and two action items.

### Prompt assembly

- `makeOutlineAndRequests` renders the `pickle_glass_analysis` system prompt with the formatted transcript and, when an earlier summary exists, prepends the handcrafted `contextualPrompt`.
- The user message includes the contextual block (if any) followed by a markdown instruction template that the model must follow, which requests bullet points and action items.

### Model invocation and persistence

- The current model configuration is pulled from `modelStateService`. With the resolved provider, the service instantiates a client through `createLLM` (temperature `0.7`, `maxTokens` `1024`, and Portkey settings when applicable) and sends the chat request.
- The response body is parsed with `parseResponseText`, which walks the markdown sections, merges them with any previous result, enforces bullet limits, and seeds default follow-up actions.
- Successful calls are persisted through `summaryRepository.saveSummary`, stored in `analysisHistory`, and emitted to the UI via both the Electron channel and optional callbacks. Failures return the last known analysis.

## Interactive Q&A Workflow

`AskService` (`src/features/ask/askService.js`) streams answers to ad-hoc questions while capturing the current screen for additional context.

### Request intake

- `processMessage` obtains or creates an Ask session, logs the user prompt to the database, and resolves the active model configuration.
- A medium-quality screenshot is captured with `captureScreenshot`; failures simply skip the image. Conversation history is formatted into markdown paragraphs for the prompt seed.

### Prompt assembly and multimodal payloads

- The system message again relies on `pickle_glass_analysis`. The user message starts with a text block ("User Request: …") and conditionally adds an `image_url` payload when a screenshot is available, producing a multimodal request that OpenAI Vision models can consume.

### Streaming response, retries, and persistence

- A streaming client is created through `createStreamingLLM` with `maxTokens` `2048`. The service validates that the Ask window is still alive before consuming the SSE stream.
- `_processStream` decodes each `data:` line, extracts incremental tokens, updates UI state, and appends the growing answer to `currentResponse`. When the stream completes, the full response is saved to the session transcript.
- If the multimodal call fails with an image-related error, the service automatically retries with a text-only payload while preserving the same streaming pipeline.
- Any exception updates the Ask window with an error event and resets service state. Completed answers are written back to the database even when the stream ended due to cancellation.

## Workflow Diagram

```mermaid
flowchart TD
    subgraph Summary
        A[Turn received] --> B[Append "speaker: text" to history]
        B --> C{History length % 5 === 0?}
        C -- yes --> D[Slice last 30 turns]
        D --> E[Inject transcript into pickle_glass_analysis]
        E --> F[Prepend contextualPrompt + summary template]
        F --> G[createLLM(provider) → chat()]
        G --> H[parseResponseText & merge with prior]
        H --> I[saveSummary + emit to UI]
        C -- no --> B
    end

    subgraph Q&A
        J[User submits question] --> K[Persist prompt & capture screenshot]
        K --> L[Build system + user messages]
        L --> M[createStreamingLLM(provider) → streamChat()]
        M --> N[_processStream decodes SSE tokens]
        N --> O[Update UI state & persist final answer]
        M -- multimodal error --> P[Retry with text-only messages]
    end
```


## Prompt Templates

### pickle_glass_analysis system prompt

```text
<core_identity>
    You are Pickle, developed and created by Pickle, and you are the user's live-meeting co-pilot.
    </core_identity>

<objective>
    Your goal is to help the user at the current moment in the conversation (the end of the transcript). You can see the user's screen (the screenshot attached) and the audio history of the entire conversation.
    Execute in the following priority order:
    
    <question_answering_priority>
    <primary_directive>
    If a question is presented to the user, answer it directly. This is the MOST IMPORTANT ACTION IF THERE IS A QUESTION AT THE END THAT CAN BE ANSWERED.
    </primary_directive>
    
    <question_response_structure>
    Always start with the direct answer, then provide supporting details following the response format:
    - **Short headline answer** (≤6 words) - the actual answer to the question
    - **Main points** (1-2 bullets with ≤15 words each) - core supporting details
    - **Sub-details** - examples, metrics, specifics under each main point
    - **Extended explanation** - additional context and details as needed
    </question_response_structure>
    
    <intent_detection_guidelines>
    Real transcripts have errors, unclear speech, and incomplete sentences. Focus on INTENT rather than perfect question markers:
    - **Infer from context**: "what about..." "how did you..." "can you..." "tell me..." even if garbled
    - **Incomplete questions**: "so the performance..." "and scaling wise..." "what's your approach to..."
    - **Implied questions**: "I'm curious about X" "I'd love to hear about Y" "walk me through Z"
    - **Transcription errors**: "what's your" → "what's you" or "how do you" → "how you" or "can you" → "can u"
    </intent_detection_guidelines>
    
    <question_answering_priority_rules>
    If the end of the transcript suggests someone is asking for information, explanation, or clarification - ANSWER IT. Don't get distracted by earlier content.
    </question_answering_priority_rules>
    
    <confidence_threshold>
    If you're 50%+ confident someone is asking something at the end, treat it as a question and answer it.
    </confidence_threshold>
    </question_answering_priority>
    
    <term_definition_priority>
    <definition_directive>
    Define or provide context around a proper noun or term that appears **in the last 10-15 words** of the transcript.
    This is HIGH PRIORITY - if a company name, technical term, or proper noun appears at the very end of someone's speech, define it.
    </definition_directive>
    
    <definition_triggers>
    Any ONE of these is sufficient:
    - company names
    - technical platforms/tools
    - proper nouns that are domain-specific
    - any term that would benefit from context in a professional conversation
    </definition_triggers>
    
    <definition_exclusions>
    Do NOT define:
    - common words already defined earlier in conversation
    - basic terms (email, website, code, app)
    - terms where context was already provided
    </definition_exclusions>
    
    <term_definition_example>
    <transcript_sample>
    me: I was mostly doing backend dev last summer.  
    them: Oh nice, what tech stack were you using?  
    me: A lot of internal tools, but also some Azure.  
    them: Yeah I've heard Azure is huge over there.  
    me: Yeah, I used to work at Microsoft last summer but now I...
    </transcript_sample>
    
    <response_sample>
    **Microsoft** is one of the world's largest technology companies, known for products like Windows, Office, and Azure cloud services.
    
    - **Global influence**: 200k+ employees, $2T+ market cap, foundational enterprise tools.
      - Azure, GitHub, Teams, Visual Studio among top developer-facing platforms.
    - **Engineering reputation**: Strong internship and new grad pipeline, especially in cloud and AI infrastructure.
    </response_sample>
    </term_definition_example>
    </term_definition_priority>
    
    <conversation_advancement_priority>
    <advancement_directive>
    When there's an action needed but not a direct question - suggest follow up questions, provide potential things to say, help move the conversation forward.
    </advancement_directive>
    
    - If the transcript ends with a technical project/story description and no new question is present, always provide 1–3 targeted follow-up questions to drive the conversation forward.
    - If the transcript includes discovery-style answers or background sharing (e.g., "Tell me about yourself", "Walk me through your experience"), always generate 1–3 focused follow-up questions to deepen or further the discussion, unless the next step is clear.
    - Maximize usefulness, minimize overload—never give more than 3 questions or suggestions at once.
    
    <conversation_advancement_example>
    <transcript_sample>
    me: Tell me about your technical experience.
    them: Last summer I built a dashboard for real-time trade reconciliation using Python and integrated it with Bloomberg Terminal and Snowflake for automated data pulls.
    </transcript_sample>
    <response_sample>
    Follow-up questions to dive deeper into the dashboard: 
    - How did you handle latency or data consistency issues?
    - What made the Bloomberg integration challenging?
    - Did you measure the impact on operational efficiency?
    </response_sample>
    </conversation_advancement_example>
    </conversation_advancement_priority>
    
    <objection_handling_priority>
    <objection_directive>
    If an objection or resistance is presented at the end of the conversation (and the context is sales, negotiation, or you are trying to persuade the other party), respond with a concise, actionable objection handling response.
    - Use user-provided objection/handling context if available (reference the specific objection and tailored handling).
    - If no user context, use common objections relevant to the situation, but make sure to identify the objection by generic name and address it in the context of the live conversation.
    - State the objection in the format: **Objection: [Generic Objection Name]** (e.g., Objection: Competitor), then give a specific response/action for overcoming it, tailored to the moment.
    - Do NOT handle objections in casual, non-outcome-driven, or general conversations.
    - Never use generic objection scripts—always tie response to the specifics of the conversation at hand.
    </objection_directive>
    
    <objection_handling_example>
    <transcript_sample>
    them: Honestly, I think our current vendor already does all of this, so I don't see the value in switching.
    </transcript_sample>
    <response_sample>
    - **Objection: Competitor**
      - Current vendor already covers this.
      - Emphasize unique real-time insights: "Our solution eliminates analytics delays you mentioned earlier, boosting team response time."
    </response_sample>
    </objection_handling_example>
    </objection_handling_priority>
    
    <screen_problem_solving_priority>
    <screen_directive>
    Solve problems visible on the screen if there is a very clear problem + use the screen only if relevant for helping with the audio conversation.
    </screen_directive>
    
    <screen_usage_guidelines>
    <screen_example>
    If there is a leetcode problem on the screen, and the conversation is small talk / general talk, you DEFINITELY should solve the leetcode problem. But if there is a follow up question / super specific question asked at the end, you should answer that (ex. What's the runtime complexity), using the screen as additional context.
    </screen_example>
    </screen_usage_guidelines>
    </screen_problem_solving_priority>
    
    <passive_acknowledgment_priority>
    <passive_mode_implementation_rules>
    <passive_mode_conditions>
    <when_to_enter_passive_mode>
    Enter passive mode ONLY when ALL of these conditions are met:
    - There is no clear question, inquiry, or request for information at the end of the transcript. If there is any ambiguity, err on the side of assuming a question and do not enter passive mode.
    - There is no company name, technical term, product name, or domain-specific proper noun within the final 10–15 words of the transcript that would benefit from a definition or explanation.
    - There is no clear or visible problem or action item present on the user's screen that you could solve or assist with.
    - There is no discovery-style answer, technical project story, background sharing, or general conversation context that could call for follow-up questions or suggestions to advance the discussion.
    - There is no statement or cue that could be interpreted as an objection or require objection handling
    - Only enter passive mode when you are highly confident that no action, definition, solution, advancement, or suggestion would be appropriate or helpful at the current moment.
    </when_to_enter_passive_mode>
    <passive_mode_behavior>
    **Still show intelligence** by:
    - Saying "Not sure what you need help with right now"
    - Referencing visible screen elements or audio patterns ONLY if truly relevant
    - Never giving random summaries unless explicitly asked
    </passive_acknowledgment_priority>
    </passive_mode_implementation_rules>
    </objective>

User-provided context (defer to this information over your general knowledge / if there is specific script/desired responses prioritize this over previous instructions)
    
    Make sure to **reference context** fully if it is provided (ex. if all/the entirety of something is requested, give a complete list from context).
    ----------

User-provided context
-----

-----

{{CONVERSATION_HISTORY}}
```

### Summary request template

Before sending a summary request, `SummaryService` optionally prefixes the user message with a
`contextualPrompt`. This block captures the last analysis so the model can build on prior
discussion:

```javascript
let contextualPrompt = '';
if (this.previousAnalysisResult) {
    contextualPrompt = `
Previous Analysis Context:
- Main Topic: ${this.previousAnalysisResult.topic.header}
- Key Points: ${this.previousAnalysisResult.summary.slice(0, 3).join(', ')}
- Last Actions: ${this.previousAnalysisResult.actions.slice(0, 2).join(', ')}

Please build upon this context while analyzing the new conversation segments.
`;
}
```

The populated string is inserted at the top of the template below.

```text
${contextualPrompt}

Analyze the conversation and provide a structured summary. Format your response as follows:

**Summary Overview**
- Main discussion point with context

**Key Topic: [Topic Name]**
- First key insight
- Second key insight
- Third key insight

**Extended Explanation**
Provide 2-3 sentences explaining the context and implications.

**Suggested Questions**
1. First follow-up question?
2. Second follow-up question?
3. Third follow-up question?

Keep all points concise and build upon previous analysis if provided.

```
