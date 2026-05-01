# 25. AI-Assisted Salesforce Development

AI coding tools (Claude Code, Codex, Copilot, Cursor, and the rest) change Salesforce work meaningfully. They also introduce new ways to destroy data, leak secrets, and ship low-quality code at high speed. This chapter is the safety brief.

The advice here is platform-aware. Salesforce has specific risks (production data in sandboxes, governor limits, AppExchange security review) that general "use AI safely" guides do not cover.

## The risks, named

### 1. Destructive operations against the wrong org

The single highest risk. AI tools, especially agentic ones with command execution, can:

- Drop a custom field that contains live data.
- Delete records via SOQL.
- Push a destructive metadata change to production by accident.
- Run a `--no-prompt` deletion the user did not approve.

This happens when the tool is connected to multiple orgs and operates on the wrong one.

### 2. Data leakage in prompts

Pasting production data into an AI tool's prompt sends it to the AI provider's servers. Includes:

- Customer PII in error messages or sample records.
- API tokens in code samples.
- Internal documentation.
- Org structure that reveals customer identity.

For some industries (healthcare, finance, regulated) this is a compliance violation.

### 3. Hallucinated metadata or APIs

AI tools sometimes invent:

- Metadata field names that don't exist.
- Apex methods that look right but fail to compile.
- REST endpoints that aren't real.
- Governor limits that are made up.

Code that passes review on aesthetic grounds may not run.

### 4. Governor limit violations at scale

AI generates code that works for a single record and silently fails the bulk case:

- Loops with SOQL inside.
- Multiple DML statements where one bulk would do.
- Recursive triggers that hit depth limits.

The AI rarely thinks in bulk by default. You have to ask.

### 5. Auth token exfiltration

If your CLI auth is stored on a machine running an AI agent with shell access, the agent can read the auth file, the SFDX auth URL, the JWT key. Anything in the AI's context window may end up in its provider's logs.

### 6. Quality drift

AI suggestions are confident. Reviewers tend to approve. Over time, the average code quality follows the AI's defaults rather than your team's standards.

## The pattern: AI suggests, human reviews, CI verifies

The safest workflow puts three independent checks between the AI's suggestion and production:

1. **AI generates** code, configuration, or commands.
2. **Human reads carefully**, especially anything destructive or that touches data.
3. **CI verifies** with static analysis, compile checks, tests, and security scans.

If any step is skipped, the protection breaks. Treating AI as a colleague that produces correct work without review is wrong. Treating AI as a tool that generates suggestions which need verification is right.

## Org isolation

The most important rule: **never connect an AI agent to production**.

In practice:

- Set up scratch orgs and sandboxes for AI experimentation.
- Authenticate the AI tool only to those orgs.
- Production credentials live in a separate, restricted environment (CI secrets manager, approved deploy host).
- The AI tool should literally not have a token that can act on production.

Concretely, in the Salesforce CLI:

```bash
# AI tool uses these
sf org login web --alias dev-scratch
sf org login web --alias integration-sandbox

# AI tool should not see these
# Production tokens live in CI/CD secrets, not on developer machines.
```

If a developer needs to run a production command, they do it from a clean shell session, not from inside the AI tool's context.

### Claude Code-specific guidance

Claude Code supports:

- Permission modes (`auto`, `plan`, `manual`).
- Allow/deny lists for commands.
- Settings to deny destructive operations.

Use them. Configure `.claude/settings.json` to:

- Deny `sf org delete`, `sf data delete`, `sf project deploy start --target-org production`.
- Require confirmation for any `sf` command targeting orgs not on an approved list.
- Block raw `curl` to Salesforce REST endpoints.

A starter:

```json
{
  "permissions": {
    "deny": [
      "Bash(sf project deploy start*--target-org production*)",
      "Bash(sf data delete*)",
      "Bash(rm -rf*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)",
      "Bash(sf org delete*)"
    ]
  }
}
```

Adapt to your environments and team.

### Codex/Copilot-specific guidance

These tools generate code in your editor. They don't directly execute. The risks shift:

- Code suggestions can include destructive patterns (e.g., `delete trigger.old`).
- Code can hallucinate APIs.
- Auto-complete can paste credentials if you've ever opened a file containing them in the same session.

Use:

- `.gitignore` and `.copilotignore` (if available) for sensitive files.
- Code review focused on what the AI generated, not just the diff.
- Prompts that ask for "bulk-safe" code by default.

### Cursor-specific guidance

Cursor wraps an editor with AI agents. Same risks as Claude Code, plus the editor-context risks of Copilot. Configure the agent's command allow/deny list. Treat agent suggestions as code review candidates, not ready-to-merge changes.

## Data handling

The rule: assume anything in your prompt can be retained by the provider for training, debugging, or moderation.

What that means:

- **Never paste real production data into a prompt.** Synthesise example data instead.
- **Strip PII before sharing schema.** "An Account with a Phone field" beats "Account where Phone = '+1-415-555-1234' for John Doe".
- **Keep secrets out.** API keys, OAuth tokens, JWT certs, anything credentialed.
- **Be cautious with org structure that reveals customer identity.** Custom field names that name a customer, sObjects with proprietary names.

### When you legitimately need to share data shape

- Use anonymised examples. Real shape, fake values.
- Use Salesforce's standard sample data when possible.
- For complex schemas, share the metadata XML (which is public schema) but strip data references.

### Provider data policies

Most AI providers offer "no-train" modes for enterprise customers:

- Claude has a setting where prompts are not used for training.
- OpenAI's enterprise tier is similar.
- GitHub Copilot for Business has stricter data handling than the consumer tier.

If your team handles regulated data, use these tiers.

## Code review for AI-generated code

The questions to ask, every time:

- **Is this real?** Does the field exist? Does the API exist? Does the method signature compile?
- **Is this bulk-safe?** Will it work for 200 records? 10,000?
- **Does it enforce CRUD/FLS?** Salesforce expects this. AI sometimes skips it.
- **Does it handle null and empty?** AI is optimistic.
- **Are there hidden references to data?** SOQL hardcoding ids, queries that assume specific records.
- **Are there security implications?** SOQL injection, XSS in LWC, callout to unexpected URLs.

A useful mental model: review AI code as if it came from a junior developer who is fast but careless. Plenty of right ideas, occasional stupid mistakes, sometimes confidently wrong.

## Specific Salesforce AI gotchas

### Apex generation

AI tends to generate:

- Apex without `with sharing` (silently bypasses sharing rules).
- SOQL in loops.
- Caught-and-swallowed exceptions (the agent says "Done" while nothing happened).
- Hardcoded record types or developer names.
- Tests that don't actually assert anything.

Ask explicitly for `with sharing`, bulkified SOQL, error handling, and meaningful asserts.

### LWC generation

AI tends to:

- Use `wire` adapters incorrectly (pre-`@wire` patterns from old Aura code).
- Mix imperative and reactive patterns.
- Generate jest tests with bad mocks.
- Forget the `@api` annotation on public properties.

Test thoroughly. LWC errors are often runtime, not compile-time.

### Metadata generation

AI tends to invent:

- Metadata XML that's almost right but missing a required tag.
- Permission set XML that references invalid components.
- Flow XML that's syntactically valid but semantically wrong.
- Validation rule formulas that don't compile.

Validate by deploying to a scratch org. If the deploy fails, the metadata is wrong.

### REST/SOQL queries

AI tends to:

- Use `LIKE` patterns that don't escape.
- Generate query strings that fail in namespaces (`Account` vs `lumin__Account`).
- Hardcode API versions.

Parameterise. Use `Database.queryWithBinds`. Check namespace handling explicitly.

## Testing AI-generated code

### Compile early

Don't let AI-generated Apex sit in a PR for hours before someone tries to deploy it. Compile to a scratch org as part of the PR check.

### Run tests fast

If your test suite is slow, AI-generated code is harder to verify. Speeding up tests pays double when AI is in the loop.

### Add property-based tests where possible

For Apex methods that have complex input shapes, generate random inputs in tests. AI-generated code often has narrow happy paths that fail on edge cases.

### LLM-as-judge for code review

A second AI can review the first AI's output, asking specific questions ("does this enforce CRUD/FLS?", "is this bulk-safe?"). Useful as a triage step before human review.

## Auditing what the AI did

Keep a paper trail:

- AI tool sessions: many tools (Claude Code, Cursor) save transcripts. Keep them, at least for changes that ship.
- Git commits: include "Generated with assistance from..." attribution if your team's policy allows. (Some teams prefer not to flag AI assistance in commits; clarify your stance.)
- Code review notes: when reviewing AI-generated code, note what you accepted vs questioned. Patterns emerge.

For regulated industries: assume an auditor will ask what was AI-generated and what was human-written. Your records should be clear.

## Org-level safeguards

Independent of the AI tool, configure your orgs to limit damage:

- **Deploy windows.** Production accepts deploys only during certain hours.
- **Approval gates.** A human approves every production deploy.
- **Validation requirement.** Production deploys must come from a validated job.
- **Backup before destructive change.** Take a data snapshot before running migrations.
- **Rollback playbook.** Know how to revert a bad change. See [Chapter 24](./24-rollback-and-hotfixes.md).

These don't directly involve AI, but they limit the blast radius when an AI-suggested change is wrong.

## Quality through AI

AI tools done well can raise quality:

- Generate tests for under-tested code.
- Write documentation faster.
- Suggest refactors.
- Catch security issues during review.
- Bulkify badly-written code.

Use them for those wins. But the wins come from a deliberate pattern (AI suggests, human reviews, CI verifies), not from accepting suggestions blindly.

## A team's policy template

Adapt this for your team:

> 1. AI tools (Claude Code, Codex, Copilot, Cursor, etc.) are allowed for development assistance.
> 2. AI tools are never authenticated to production. Production credentials live in CI/CD secrets only.
> 3. Production data is never pasted into AI prompts. Synthesised examples only.
> 4. AI-generated code is reviewed by a human before merge. Code review questions explicitly cover bulk-safety, CRUD/FLS enforcement, and security.
> 5. AI-generated metadata is validated against a scratch org before merge.
> 6. Use enterprise/no-train tiers of AI providers when handling sensitive data.
> 7. Suspicious or destructive AI suggestions are escalated to the team rather than approved silently.
> 8. Audit logs and AI tool transcripts are retained for the duration of the project.

Sign it as a team. Revisit quarterly.

## When AI is the wrong tool

A few situations where AI assistance is more risk than reward:

- Production hotfixes under time pressure. Mistakes get amplified.
- Migrations involving customer data. The cost of a bad query is too high.
- Compliance-sensitive code. Audit trail of "AI generated this" is harder to defend.
- Code reviews of AI-generated code by another AI as the only check. Two AI tools agreeing does not constitute review.

For those, fall back to careful human-driven work.

## When AI is genuinely useful

- Boilerplate generation: test classes, permission sets, validation rules.
- Refactoring within established patterns.
- Documentation drafts.
- Translation between styles (Apex to LWC patterns, old to new APIs).
- Test data generation for synthetic scenarios.
- Investigating unfamiliar code: "explain this trigger".
- Writing conversation copy (Agentforce specifically benefits).

Lean on AI for these. Be careful with everything else.

## A short checklist before merging AI code

- [ ] Does it compile? Ran in a scratch org?
- [ ] Tests written? Tests assert real behaviour?
- [ ] CRUD/FLS enforced where needed?
- [ ] Bulk-safe? Tested with 200 records?
- [ ] Sharing model intentional? `with sharing` or `without sharing` deliberate?
- [ ] No hardcoded ids?
- [ ] No swallowed exceptions?
- [ ] Security: no SOQL injection, no XSS, no exposed secrets?
- [ ] Pattern-consistent with the rest of the codebase?
- [ ] Reviewed by a human who understands what's deployed?

If any answer is "no" or "not sure", do not merge.

## References

- [Salesforce Code Analyzer](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/)
- [PMD Apex rules](https://docs.pmd-code.org/latest/pmd_rules_apex.html)
- [AppExchange Security Review](https://partners.salesforce.com/s/education/general/Security_Review)
- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Apex security best practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_security.htm)
- [Anthropic enterprise data privacy](https://www.anthropic.com/legal/commercial-terms)
- [GitHub Copilot data handling](https://docs.github.com/en/copilot/about-github-copilot/about-github-copilot-business)
