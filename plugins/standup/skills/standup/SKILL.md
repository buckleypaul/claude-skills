---
name: standup
description: Generate daily standup reports by aggregating activity from Slack, Jira, Confluence, GitHub, and Gmail
---

# Standup Report Generator

Generate comprehensive daily standup reports by aggregating your activity from multiple sources.

## Output Format

```markdown
# Standup - YYYY-MM-DD

**Done**
- Item 1
- Item 2

**Doing**
- Item 1
- Item 2

**Blockers**
- Item 1 (or "None")
```

Output file: `~/standups/standup-YYYY-MM-DD.md`

---

## Workflow

Execute these phases in order:

### Phase 1: Calculate Date Range

Parse the `days_back` argument from the user input. If not provided, default to 1 day.

```
days_back = {argument} or 1
end_date = today (YYYY-MM-DD format)
start_date = today - days_back days (YYYY-MM-DD format)
```

Display to user:
```
Generating standup report for: {end_date}
Looking back {days_back} day(s) to: {start_date}
```

### Phase 2: Parallel Data Collection

Use the Task tool to spawn 5 sub-agents in parallel. Each agent should:
1. Use ToolSearch to load required MCP tools
2. Query the data source
3. Return categorized items (done/doing/blockers)

**IMPORTANT: Launch all 5 agents in a SINGLE message with multiple Task tool calls.**

#### Slack Sub-Agent

```
Prompt: |
  Search Slack for my messages from {start_date} to {end_date}.

  1. Use ToolSearch to load: slack_search_public_and_private
  2. Search with query: "from:me after:{start_date}"
  3. Apply filtering rules:
     - EXCLUDE: DM channels (1:1 conversations), messages < 20 chars, bot messages
     - INCLUDE: Substantive messages (>50 chars), messages with links/attachments
  4. Categorize each message:
     - Done: Past tense, completed work mentions
     - Doing: Present/future tense, ongoing work
     - Potential Blockers: Keywords like "blocked", "waiting on", "stuck", "need help"

  Return JSON:
  {
    "done": ["item1", "item2"],
    "doing": ["item1"],
    "blockers": ["item1"],
    "source": "slack"
  }
```

#### Jira Sub-Agent

```
Prompt: |
  Search Jira for my issues updated from {start_date} to {end_date}.

  1. Use ToolSearch to load: searchJiraIssuesUsingJql
  2. Search with JQL: assignee = currentUser() AND updated >= "{start_date}"
  3. For each issue, fetch status and transition history
  4. Categorize:
     - Done: Status changed TO "Done", "Closed", or "Resolved" during date range
     - Doing: Current status is "In Progress" or "To Do"
     - Blockers: Status is "Blocked" OR has blocker flag/label
  5. Format items as: "[PROJ-123] Issue title"

  Return JSON:
  {
    "done": ["[PROJ-123] Completed feature X", "[PROJ-456] Fixed bug Y"],
    "doing": ["[PROJ-789] Working on feature Z"],
    "blockers": ["[PROJ-111] Blocked on API access"],
    "source": "jira"
  }
```

#### Confluence Sub-Agent

```
Prompt: |
  Search Confluence for pages I contributed to from {start_date} to {end_date}.

  1. Use ToolSearch to load: searchConfluenceUsingCql
  2. Search with CQL: contributor = currentUser() AND lastModified >= "{start_date}"
  3. For each page, check if it's a draft or published
  4. Categorize:
     - Done: Published/complete pages
     - Doing: Draft pages, recently edited but not published
  5. Format items as: "Page title" with link if available

  Return JSON:
  {
    "done": ["Published design doc for Feature X"],
    "doing": ["Drafting architecture proposal"],
    "blockers": [],
    "source": "confluence"
  }
```

#### GitHub Sub-Agent

```
Prompt: |
  Search GitHub for my activity from {start_date} to {end_date}.

  1. Use Bash with gh CLI to run these commands:
     - gh pr list --author @me --state all --search "created:>={start_date}"
     - gh pr list --search "reviewed-by:@me created:>={start_date}"
     - Check for PRs with requested changes or stale reviews

  2. Categorize:
     - Done: Merged PRs, completed reviews
     - Doing: Open PRs awaiting review, PRs with feedback to address
     - Blockers: PRs with "changes requested", stale PRs awaiting review >2 days

  3. Aggregation rules:
     - Multiple PR reviews -> single line: "Reviewed 3 PRs: [#123], [#456], [#789]"
     - List individual PRs authored

  Return JSON:
  {
    "done": ["Merged PR #123: Add user auth", "Reviewed 3 PRs: #456, #789, #101"],
    "doing": ["PR #234: Implement caching (awaiting review)"],
    "blockers": ["PR #345: Changes requested, waiting on design clarification"],
    "source": "github"
  }
```

#### Gmail Sub-Agent

```
Prompt: |
  Search Gmail for emails I sent from {start_date} to {end_date}.

  1. Use ToolSearch to load Gmail MCP tools (e.g., gmail_search, gmail_list_messages)
  2. Search for sent emails with query: "from:me after:{start_date} before:{end_date}"
  3. Apply filtering rules:
     - EXCLUDE: Auto-replies, calendar invites, newsletters, internal notifications
     - EXCLUDE: Short replies (<50 chars) like "Thanks", "Got it", "LGTM"
     - INCLUDE: Substantive emails with updates, decisions, or deliverables
     - INCLUDE: Emails with attachments (likely sharing documents/work products)
  4. Categorize each email:
     - Done: Emails sharing completed work, status updates, deliverables sent
     - Doing: Emails discussing ongoing work, requests for feedback on drafts
     - Potential Blockers: Emails asking for help, escalations, waiting on responses
  5. Format items as: "Sent: {brief subject/description}"

  Return JSON:
  {
    "done": ["Sent: Q4 report to stakeholders", "Sent: Design review feedback to team"],
    "doing": ["Sent: Draft proposal to manager for review"],
    "blockers": ["Sent: Escalation to vendor - awaiting response"],
    "source": "gmail"
  }
```

### Phase 3: Manual Input Collection

After parallel agents complete, prompt user for additional sources that are stubbed:

**Calendar (stub):**
```
I don't have access to your calendar. Please share notable meetings you led or presented in:
- Meetings where you drove discussion
- Presentations or demos you gave
- EXCLUDE: 1:1s, routine standups, meetings where you were just an attendee

Enter items (one per line, empty line to finish):
```

Use AskUserQuestion for each prompt with a text input option.

### Phase 4: Aggregation & Deduplication

Merge all collected items and deduplicate:

1. **Merge by identifier:**
   - Jira: Same ticket key (PROJ-123)
   - GitHub: Same PR number (#123)
   - Confluence: Same page ID/title

2. **Aggregate PR reviews:**
   - If multiple PR reviews, combine: "Reviewed N PRs: [#A], [#B], [#C]"

3. **Apply content rules:**
   - Use factual, professional language
   - Remove boastful phrases ("crushed it", "knocked it out of the park", "nailed it")
   - Ignore participation-only items ("joined discussion", "attended meeting")
   - Exclude 1:1s and routine standups
   - Convert vague items to specific outcomes where possible

4. **Combine related items:**
   - If Slack message references a Jira ticket already captured, merge context
   - If multiple updates to same Confluence page, combine into one item

### Phase 5: One-by-One Confirmation

For each aggregated item, present to user for confirmation:

```
Item 1 of N [Source: Jira]
"[PROJ-123] Implemented user authentication feature"

Category: Done

Options:
[K] Keep as-is
[E] Edit text
[R] Remove
[M] Move to different category (Done/Doing/Blockers)
```

Use AskUserQuestion with these options for each item.

If user selects:
- **Keep**: Add to final list
- **Edit**: Prompt for edited text, then add
- **Remove**: Skip this item
- **Move**: Prompt for new category, then add to that category

### Phase 6: Final Questions

After all items are confirmed, ask two final questions:

**Question 1: Missed Items**
```
Are there any work items I missed that you'd like to add?

Enter additional items (one per line, specify category with prefix):
- DONE: Item description
- DOING: Item description
- BLOCKER: Item description

(Empty line to finish)
```

**Question 2: Additional Blockers**
```
Are there any blockers I should know about?
- Technical blockers
- Waiting on someone/something
- Resource constraints
- External dependencies

(Empty line if none)
```

**Question 3: Jira Ticket Recommendations**

After collecting all items, identify work that lacks Jira tickets:
- Work items mentioned in Slack/GitHub without ticket references
- Manual items added without ticket numbers

For each untracked item, ask:
```
This work item doesn't have a Jira ticket:
"{item description}"

Would you like me to recommend creating a ticket?
[Y] Yes, recommend a ticket
[N] No, skip
[A] Already tracked elsewhere
```

If yes, suggest ticket details:
- Summary
- Type (Task/Story/Bug)
- Suggested description

EXCLUDE from ticket recommendations:
- Meetings and presentations
- PR reviews
- Customer questions and support
- Purely responsive work

### Phase 7: Generate Output

1. **Create output directory:**
```bash
mkdir -p ~/standups
```

2. **Generate standup file:**

Filename: `~/standups/standup-{YYYY-MM-DD}.md`

Content:
```markdown
# Standup - {YYYY-MM-DD}

**Done**
- {item 1}
- {item 2}
- {item 3}

**Doing**
- {item 1}
- {item 2}

**Blockers**
- {item 1 or "None"}
```

3. **Write file using Write tool**

4. **Display final standup to user:**
```
Standup saved to: ~/standups/standup-{date}.md

---
{display full standup content}
---

You can copy this to your standup meeting or share with your team.
```

---

## Blocker Detection Rules

Automatically flag as potential blockers:

**Jira:**
- Status = "Blocked"
- Has label: "blocker", "blocked", "impediment"
- Blocker link type present

**Slack:**
- Keywords: "blocked", "waiting on", "stuck", "need help", "can't proceed", "dependency"
- Questions without responses after 24h

**GitHub:**
- PRs with "changes requested" status
- PRs with review requested >2 days ago with no response
- CI/CD failures blocking merge

**Gmail:**
- Emails with keywords: "blocked", "waiting on", "need approval", "urgent", "escalation"
- Sent emails without responses after 48h (potential waiting on someone)
- Emails to external vendors/partners requesting action

---

## Content Rules

**DO:**
- Use factual, professional language
- Be specific about what was accomplished
- Include ticket/PR numbers as references
- Aggregate related items with links

**DON'T:**
- Use boastful language ("crushed it", "knocked it out")
- Include participation-only items
- List meetings you merely attended
- Include 1:1s or routine standups
- Use vague language ("worked on stuff")

**Examples:**

Bad: "Crushed the auth feature, it's amazing!"
Good: "[PROJ-123] Completed user authentication implementation"

Bad: "Attended team meeting"
Good: (exclude this - participation only)

Bad: "Reviewed some PRs"
Good: "Reviewed 3 PRs: #456, #789, #101"

---

## Error Handling

If a data source fails:
1. Log the error
2. Continue with other sources
3. Inform user: "Note: Could not fetch data from {source}: {error}"
4. Ask user to manually provide items from that source

If no data is found:
1. Inform user: "No activity found in {source} for this date range"
2. Continue with other sources
3. Still generate standup with available data

---

## Verification Checklist

Before finalizing, verify:
- [ ] All sources were queried (or errors logged)
- [ ] Duplicate items removed
- [ ] Items properly categorized
- [ ] No boastful language
- [ ] Blockers section populated or shows "None"
- [ ] Output file created successfully
- [ ] Date format is correct (YYYY-MM-DD)
