# Chapter 5: Patterns and Troubleshooting

These patterns emerged from skills created by early adopters and internal teams. They represent common approaches we've seen work well, not prescriptive templates.

## Choosing Your Approach: Problem-First vs. Tool-First

Think of it like Home Depot. You might walk in with a problem — "I need to fix a kitchen cabinet" — and an employee points you to the right tools. Or you might pick out a new drill and ask how to use it for your specific job.

Skills work the same way:

- **Problem-first**: "I need to set up a project workspace" → Your skill orchestrates the right MCP calls in the right sequence. Users describe outcomes; the skill handles the tools.
- **Tool-first**: "I have Notion MCP connected" → Your skill teaches Claude the optimal workflows and best practices. Users have access; the skill provides expertise.

Most skills lean one direction. Knowing which framing fits your use case helps you choose the right pattern below.

---

## Pattern 1: Sequential Workflow Orchestration

**Use when:** Your users need multi-step processes in a specific order.

**Example structure:**

```markdown
## Workflow: Onboard New Customer

### Step 1: Create Account
Call MCP tool: `create_customer`
Parameters: name, email, company

### Step 2: Setup Payment
Call MCP tool: `setup_payment_method`
Wait for: payment method verification

### Step 3: Create Subscription
Call MCP tool: `create_subscription`
Parameters: plan_id, customer_id (from Step 1)

### Step 4: Send Welcome Email
Call MCP tool: `send_email`
Template: welcome_email_template
```

**Key techniques:**

- Explicit step ordering
- Dependencies between steps
- Validation at each stage
- Rollback instructions for failures

---

## Pattern 2: Multi-MCP Coordination

**Use when:** Workflows span multiple services.

**Example: Design-to-development handoff**

```markdown
### Phase 1: Design Export (Figma MCP)
1. Export design assets from Figma
2. Generate design specifications
3. Create asset manifest

### Phase 2: Asset Storage (Drive MCP)
1. Create project folder in Drive
2. Upload all assets
3. Generate shareable links

### Phase 3: Task Creation (Linear MCP)
1. Create development tasks
2. Attach asset links to tasks
3. Assign to engineering team

### Phase 4: Notification (Slack MCP)
1. Post handoff summary to #engineering
2. Include asset links and task references
```

---

## Pattern 3: Iterative Refinement

**Use when:** Output quality improves with iteration.

**Example: Report generation**

```markdown
## Iterative Report Creation

### Initial Draft
1. Fetch data via MCP
2. Generate first draft report
3. Save to temporary file

### Quality Check
1. Run validation script: `scripts/check_report.py`
2. Identify issues:
   - Missing sections
   - Inconsistent formatting
   - Data validation errors

### Refinement Loop
1. Address each identified issue
2. Regenerate affected sections
3. Re-validate
4. Repeat until quality threshold met
```

---

## Pattern 4: Conditional Branching

**Use when:** Different inputs require different workflows.

```markdown
## Handle Support Ticket

### Classify Ticket
Determine ticket type from description:
- Bug report → Bug Workflow
- Feature request → Feature Workflow
- Question → Knowledge Base Workflow

### Bug Workflow
1. Search existing issues (Linear MCP)
2. If duplicate: link and close
3. If new: create bug report with reproduction steps

### Feature Workflow
1. Check product roadmap (Notion MCP)
2. Create feature request
3. Notify product team (Slack MCP)

### Knowledge Base Workflow
1. Search documentation (Confluence MCP)
2. Generate response from relevant docs
3. If no docs found: escalate to human
```

---

## Pattern 5: Template-Based Generation

**Use when:** Outputs follow consistent formats.

```markdown
## Generate API Documentation

### Template
For each endpoint, generate:

**{METHOD} {path}**

Description: {description}

Parameters:
| Name | Type | Required | Description |
|------|------|----------|-------------|
| {name} | {type} | {required} | {desc} |

Response:
\`\`\`json
{example_response}
\`\`\`

### Instructions
1. Fetch API schema from MCP
2. For each endpoint, fill template
3. Add authentication section
4. Generate table of contents
```

---

## Troubleshooting

### Skill Not Triggering

**Symptom:** Claude doesn't use the skill when expected.

**Common causes:**

1. **Trigger mismatch** — User phrasing doesn't match skill triggers
   - Add more trigger variations
   - Test with different phrasings
   - Consider adding a slash command trigger

2. **Competing skills** — Another skill matches first
   - Check skill priority/ordering
   - Make triggers more specific
   - Consider combining overlapping skills

3. **Missing context** — Skill needs information not provided
   - Add clear input requirements
   - Provide default values where possible

### MCP Integration Issues

**Symptom:** Skill fails when calling MCP tools.

**Debugging steps:**

1. Check MCP connection status
   - Verify MCP server is running
   - Check authentication/tokens

2. Verify tool availability
   - List available MCP tools
   - Confirm tool names match skill references

3. Test MCP independently
   - Ask Claude to call MCP directly (without skill)
   - "Use [Service] MCP to fetch my projects"
   - If this fails, issue is MCP not skill

4. Verify tool names
   - Skill references correct MCP tool names
   - Check MCP server documentation
   - Tool names are case-sensitive

### Output Quality Issues

**Symptom:** Skill produces inconsistent or low-quality results.

**Solutions:**

1. **Add examples** — Show Claude what good output looks like
2. **Add constraints** — Be explicit about format, length, style
3. **Add validation** — Include self-check steps in the workflow
4. **Model "laziness"** — Add explicit encouragement:

```markdown
## Performance Notes
- Take your time to do this thoroughly
- Quality is more important than speed
- Do not skip validation steps
```

> Note: Adding this to user prompts is more effective than in SKILL.md

### Large Context Issues

**Symptom:** Skill seems slow or responses degraded.

**Causes:**

- Skill content too large
- Too many skills enabled simultaneously
- All content loaded instead of progressive disclosure

**Solutions:**

1. **Optimize SKILL.md size**
   - Move detailed docs to `references/`
   - Link to references instead of inline
   - Keep SKILL.md under 5,000 words

2. **Reduce enabled skills**
   - Evaluate if you have more than 20–50 skills enabled simultaneously
   - Recommend selective enablement
   - Consider skill "packs" for related capabilities
