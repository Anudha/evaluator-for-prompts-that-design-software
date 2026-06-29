**Software Generation Prompt Evaluation Framework**

These evaluations should be run in order. Each one is a question with a pass/fail test. A prompt that fails any Category 1 evaluation should be revised before code is written.

**Category 1 — Feasibility (must pass before writing code)**

1.1 Real-world target compatibility
For each external system, API, or website the software interacts with: does the proposed approach actually work against how that system behaves in production?
Test: name the approach, name the target, state one known fact about the target that confirms or kills the approach.
Fail example: "scrape career pages with requests.get" + "Stripe uses React-rendered job listings"
1.2 Runtime environment consistency
Are the tools used across components consistent with the runtime already established?
Test: list every tool/library used per component. If the system already has Tool X running, flag any component that reimplements Tool X's job with something inferior.
Fail example: CDP browser already running for form filling, static HTTP fetcher used for crawling the same sites
1.3 Code generation timing
If any code generates other code: are all values substituted at the right time? Separate build-time constants from runtime variables explicitly.
Test: for every variable appearing in a generated file, state when it gets its value — build time or runtime. Any runtime variable inside a build-time f-string is a bug.
1.4 Platform and version constraints
Are there known version-specific behaviors of the target platform that affect the implementation?
Test: for each external dependency, name its version or version range and any known breaking changes relevant to the approach.
Fail example: Chrome 111+ requires --remote-allow-origins for CDP WebSocket connections
1.5 Network failure isolation
For every external HTTP call: what happens to the rest of the system if it fails, hangs, or returns garbage?
Test: trace each network call to its failure mode. A crash in one company's crawler should not kill the agent thread processing all other companies.

**Category 2 — Architecture (catch structural problems early)**
2.1 Single responsibility per component
Does each module do exactly one thing? Flag any module that both discovers data and acts on it, or both parses and stores.
2.2 State ownership
Where does mutable shared state live? Is there exactly one owner? Flag any state that is read and written from multiple threads without a clear ownership model.
2.3 Blocking vs non-blocking
Which operations block the main thread? Is that acceptable? Flag any long-running operation (LLM call, network fetch, browser navigation) that would freeze the UI or API if called synchronously.
2.4 Data flow traceability
Can you trace a single unit of data (e.g. one job URL) from entry point to final storage, naming every transformation? Any gap in that trace is an unspecified component.
2.5 Failure propagation
If component B depends on component A's output: what does B do when A returns nothing, returns malformed data, or throws? Specify this explicitly for every dependency edge.

**Category 3 — External system assumptions**
3.1 Authentication requirements
Does any target site require login to reach the relevant content? If yes, the approach must account for it or explicitly exclude those targets.
3.2 Anti-automation measures
Does any target site use CAPTCHA, bot detection, rate limiting, or JavaScript challenges? State what the system does in each case — skip, pause, ask human — and confirm no bypass is attempted.
3.3 DOM and rendering assumptions
For any web scraping or automation: is the target content available in static HTML, or does it require JavaScript execution to render? Name which for each target.
This is the question that would have caught the crawler problem before any code was written.
3.4 Schema stability
Does the approach depend on a specific DOM structure, API response shape, or URL pattern that could change? Flag any component that would silently break if a third-party site changed its markup.
3.5 Terms of service
Does the approach comply with each target site's terms? Flag scraping, automation, or data collection that may conflict.

**Category 4 — UI and polling behavior**
4.1 Poll side effects
For any polling loop: does the handler modify UI state beyond updating content? Switching active tabs, scrolling, or stealing focus on every poll cycle will make the UI unusable.
Test: list every DOM mutation the poll handler performs. Any mutation that affects user navigation is a bug.
4.2 First-time vs subsequent renders
For any data that arrives asynchronously: distinguish between "show this for the first time" (may switch tab, scroll into view) and "update this silently" (content only, no navigation).
4.3 User control preservation
Can the user interrupt, navigate away from, or override any agent action at any time? Flag any loop or state machine that does not yield to user input.

**Category 5 — LLM integration**
5.1 LLM output scope
What is the LLM allowed to return — structured JSON, natural language, executable code? State this explicitly. Any prompt that could return executable code that gets eval'd is a security boundary violation.
5.2 Validation before action
Is every LLM output validated against a schema before it causes a side effect? Flag any path where raw LLM output directly drives a DOM action, file write, or network request.
5.3 Hallucination surface
Which fields or values could the LLM plausibly invent if uncertain? For each: is there a deterministic check that catches fabricated values before they are used?
Fail example: LLM guesses job titles from UUID path segments with no validation that the title matches real text on the page.
5.4 Sensitive field handling
List every field the LLM is not permitted to generate answers for. Confirm each has a hard-coded skip rule, not a prompt instruction. Prompt instructions can be ignored; code cannot.
5.5 Fallback on LLM failure
For every LLM call: what happens on timeout, invalid JSON, or empty response? Is there a retry with a repair prompt, a fallback to human review, or a crash? State which.

**Category 6 — Privacy and data handling**
6.1 Data boundary
Name every piece of user data (resume, profile, email, phone). For each: confirm it never leaves the local machine except through explicitly approved submission actions.
6.2 Log content
What goes into log files? Confirm no credential, resume text, or PII is written to logs unintentionally.
6.3 Screenshot content
Screenshots capture whatever is on screen. Confirm the review flow makes clear to the user what is being captured and stored.


**The goal is that by the time code generation starts, every evaluation passes and the implementer has no architectural decisions left to make, only implementation decisions.**
