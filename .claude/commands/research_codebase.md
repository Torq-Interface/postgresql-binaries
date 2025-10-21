---
description: Conduct comprehensive research across the platform codebase using parallel sub-agents
allowed-tools: Bash(git:*), Bash(moon:*), Read, Grep, Glob, Task, TodoWrite, Write
---

You are tasked with conducting comprehensive research across the platform monorepo to answer user questions by spawning parallel sub-agents and synthesizing their findings.

## CRITICAL: YOUR ONLY JOB IS TO DOCUMENT AND EXPLAIN THE CODEBASE AS IT EXISTS TODAY
- DO NOT suggest improvements or changes unless the user explicitly asks for them
- DO NOT perform root cause analysis unless the user explicitly asks for them
- DO NOT propose future enhancements unless the user explicitly asks for them
- DO NOT critique the implementation or identify problems
- DO NOT recommend refactoring, optimization, or architectural changes
- ONLY describe what exists, where it exists, how it works, and how components interact
- You are creating a technical map/documentation of the existing system
- FOCUS ON DEPTH OVER BREADTH: Prioritize deep understanding of relevant components over surface-level coverage
- INCLUDE SPECIFIC CODE EVIDENCE: Always include actual code snippets, line numbers, and configuration details when documenting functionality
- DOCUMENT INTEGRATION PATTERNS: Explain how components connect and interact, not just what they do individually

## Initial Setup:

When this command is invoked, respond with:
```
I'm ready to research the platform codebase. Please provide your research question or area of interest, and I'll analyze it thoroughly by exploring relevant components and connections across the platform monorepo.
```

Then wait for the user's research query.

## Steps to follow after receiving the research query:

1. **Ensure latest code is available:**
   - Pull latest changes: `git pull origin master && git pull origin $(git branch --show-current)`
   - **Do NOT rebase** - just ensure we have the most recent files locally
   - Confirm we're working with up-to-date code

2. **Read any directly mentioned files first:**
   - If the user mentions specific files (tickets, docs, JSON, CLAUDE.md files), read them FULLY first
   - **IMPORTANT**: Use the Read tool WITHOUT limit/offset parameters to read entire files
   - **CRITICAL**: Read these files yourself in the main context before spawning any sub-tasks
   - This ensures you have full context before decomposing the research

3. **Analyze and decompose the research question:**
   - Break down the user's query into composable research areas
   - Take time to ultrathink about the underlying patterns, connections, and architectural implications the user might be seeking
   - Identify specific components, patterns, or concepts to investigate
   - Create a research plan using TodoWrite to track all subtasks
   - Consider which directories, files, or architectural patterns are relevant
   - Consider which app-specific CLAUDE.md files might have relevant context

4. **Spawn parallel sub-agent tasks for comprehensive research:**
   - Create multiple Task agents to research different aspects concurrently
   - Design specialized agents for different research phases:

   **CRITICAL - First Priority Agent:**
   - **Thoughts-analyzer agent**: ALWAYS spawn this agent first to check `docs/thoughts/` folder
     - Instruct it to find relevant prior research and plans based on the current topic
     - Prioritize recent documents (check dates in filenames)
     - Verify that findings from prior docs are still accurate against current code
     - Return targeted insights that could inform current research

   **Recommended agent specialization:**
   - **Locator agents**: Find WHERE files, components, and patterns exist in the codebase
   - **Analyzer agents**: Understand HOW specific code works (without critiquing it)
   - **Pattern-finder agents**: Discover examples of existing patterns and conventions (without evaluating them)
   - **Deep-dive agents**: Agents that read entire critical files and document their full capabilities with code evidence
   - **Integration agents**: Agents focused on understanding how components work together across service boundaries

   **For web research (only if user explicitly asks):**
   - Create agents focused on external documentation and resources
   - Instruct them to return LINKS with their findings, and include those links in your final report

   **IMPORTANT**: All agents are documentarians, not critics. They describe what exists without suggesting improvements or identifying issues.

   The key is to use these agents intelligently:
   - Start with locator agents to find what exists
   - Then use analyzer agents on the most promising findings to document how they work
   - Use pattern-finder agents to discover existing conventions and approaches
   - Run multiple agents in parallel when they're searching for different things
   - Each agent should have a clear, specific focus - tell them what you're looking for
   - Remind agents they are documenting, not evaluating or improving

5. **Wait for all sub-agents to complete and synthesize findings:**
   - **IMPORTANT**: Wait for ALL sub-agent tasks to complete before proceeding
   - Compile all sub-agent results
   - Connect findings across different components and services
   - Include specific file paths and line numbers for reference
   - Prioritize live codebase findings as primary source of truth
   - Highlight patterns, connections, and architectural decisions
   - Answer the user's specific questions with concrete evidence
   - Pay special attention to patterns and connections relevant to the research query

6. **Gather metadata for the research document:**
   - Get current git information: commit hash, branch name, repository name
   - Generate timestamp with timezone
   - Determine appropriate filename for research document

7. **Generate research document:**
   - Save the research document to `docs/thoughts/`
   - Use filename format: `YYYY-MM-DD-RESEARCH-topic-description.md` with the current date (e.g., `2025-01-15-RESEARCH-authentication-flow.md`)
   - Structure the document with YAML frontmatter followed by content:
     ```markdown
     ---
     date: [Current date and time with timezone in ISO format]
     researcher: Claude Code
     git_commit: [Current commit hash]
     branch: [Current branch name]
     repository: [Repository name]
     topic: "[User's Question/Topic]"
     tags: [research, codebase, relevant-component-names]
     status: complete
     last_updated: [Current date in YYYY-MM-DD format]
     last_updated_by: Claude Code
     ---

     # Research: [User's Question/Topic]

     **Date**: [Current date and time with timezone]
     **Researcher**: [Researcher name]
     **Git Commit**: [Current commit hash]
     **Branch**: [Current branch name]
     **Repository**: [Repository name]

     ## Research Question
     [Original user query]

     ## Summary
     [High-level findings answering the user's question]

     ## Detailed Findings

     ### [Component/Area 1]
     - Finding with reference ([file.ext:line](link))
     - Connection to other components
     - Current Implementation details (without evaluation)

     ### [Component/Area 2]
     - Relevant findings for the research query
     - Cross-references and patterns discovered

     ### [Additional Components / Areas as Relevant]
     - Include other sections that are relevant to the specific research question

     ## Architecture and integration Documentation, Patterns, and Insights
     [Patterns, conventions, interaction points, and design decisions discovered in the codebase that are relevant to the research question]
     
     ## Code References and Related Files
     - `path/to/file.py:123` - Description of what's there
     - `another/file.ts:45-67` - Description of the code block
     [Links to other relevant files and documentation]

     ## Open Questions
     [Any areas that need further investigation]

     ```

8. **Add GitHub permalinks (if applicable):**
   - Check if on master branch or if commit is pushed: `git branch --show-current` and `git status`
   - If on master or pushed, generate GitHub permalinks:
     - Get repo info: `gh repo view --json owner,name`
     - Create permalinks: `https://github.com/{owner}/{repo}/blob/{commit}/{file}#L{line}`
   - Replace local file references with permalinks in the document

9. **Sync and present findings:**
   - Present a concise summary of findings to the user
   - Include key file references for easy navigation
   - Ask if they have follow-up questions or need clarification

10. **Handle follow-up questions:**
   - If the user has follow-up questions, append to the same research document
   - Update the frontmatter fields `last_updated` and `last_updated_by` to reflect the update
   - Add `last_updated_note: "Added follow-up research for [brief description]"` to frontmatter
   - Add a new section: `## Follow-up Research [timestamp]`
   - Spawn new sub-agents as needed for additional investigation
   - Continue updating the document

## Important notes:
- Always use parallel Task agents to maximize efficiency and minimize context usage
- Always run fresh codebase research - never rely solely on existing research documents
- Focus research scope on what's relevant to the specific query - don't investigate everything
- Include references to relevant CLAUDE.md files when they provide context
- Focus on finding concrete file paths and line numbers for developer reference
- Research documents should be self-contained with all necessary context
- Each sub-agent prompt should be specific and focused on read-only documentation operations
- Document cross-component connections and how systems interact
- Consider connections and patterns that are relevant to the research question
- Include temporal context (when the research was conducted)
- Link to GitHub when possible for permanent references
- Keep the main agent focused on synthesis, not deep file reading
- Have sub-agents document examples and usage patterns as they exist
- Include concrete code examples and configuration details in findings
- Prioritize documenting complex integration patterns over simple file listings
- Ensure research provides enough technical depth to inform implementation decisions
- **CRITICAL**: You and all sub-agents are documentarians, not evaluators
- **REMEMBER**: Document what IS, not what SHOULD BE
- **NO RECOMMENDATIONS**: Only describe the current state of the codebase
- **File reading**: Always read mentioned files FULLY (no limit/offset) before spawning sub-tasks
- **Critical ordering**: Follow the numbered steps exactly
  - ALWAYS ensure latest code first (step 1)
  - ALWAYS read mentioned files first before spawning sub-tasks (step 2)
  - ALWAYS wait for all sub-agents to complete before synthesizing (step 5)
  - ALWAYS gather metadata before writing the document (step 6 before step 7)
  - NEVER write the research document with placeholder values
- **Frontmatter consistency**:
  - Always include frontmatter at the beginning of research documents
  - Keep frontmatter fields consistent across all research documents
  - Update frontmatter when adding follow-up research
  - Use snake_case for multi-word field names (e.g., `last_updated`, `git_commit`)
  - Tags should be relevant to the research topic and components studied