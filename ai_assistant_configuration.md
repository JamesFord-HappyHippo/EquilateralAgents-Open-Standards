# AI Assistant Configuration Standards

**Configuring AI coding assistants for automatic standards enforcement**

As AI coding assistants become integral to development workflows, configuring them to follow your standards automatically becomes essential. This document provides principles for creating effective "personality prompts" that guide AI behavior.

---

## Why Configure AI Assistants?

### The Problem

**Without configuration:**
- AI assistants reinvent solutions that already exist
- Standards violations get introduced in generated code
- Developers must manually check every suggestion
- Team conventions are inconsistent
- Learned patterns get forgotten

**With configuration:**
- AI automatically follows project standards
- Violations caught before code is written
- References existing patterns instead of creating new ones
- Consistent quality across all developers
- Institutional knowledge preserved

---

## 1. Use "Personality Prompts"

### Principle

**Configure AI assistants with context about your project's standards, conventions, and war stories.**

Think of it as onboarding a new team member—except the AI reads it every time, never forgets, and applies it consistently.

### Why This Matters

- **Consistency** - Every developer (and AI) follows same standards
- **Speed** - AI references existing patterns automatically
- **Quality** - Standards enforced on every code generation
- **Knowledge Transfer** - New developers see standards in action

### What to Include

**1. Why Standards Exist (War Stories)**

Don't just list rules—explain why they exist:

```markdown
### "No Mocks, No Fallback Data"
**Why:** We spent 5 billion tokens debugging a TypeScript conversion where
mocks hid real failures. Tests passed. Production failed.
**Rule:** Fail fast in dev, fail loud. Never hide integration failures.
```

**Why war stories work:**
- Makes rules memorable
- Shows consequences, not just theory
- Helps AI understand intent, not just syntax
- Creates pattern-matching templates

**2. Mandatory Workflow (Step-by-Step)**

Give AI a checklist to follow:

```markdown
## Before ANY Code Change:

1. **Check Standards** - Does a pattern exist? (.standards/)
2. **Follow Existing Patterns** - Use established helpers/utilities
3. **Design Errors First** - Error responses before happy path
4. **Show Your Work** - Document what you checked

[Standards checked: development_principles.md]
[Pattern found: Error handling with specific codes]
[Decision: Using existing pattern]
```

**3. Banned Patterns (With Examples)**

Show what NOT to do:

```markdown
### ❌ Mock Data or Fallbacks
// WRONG - hides real failures
if (apiCallFails) return mockData;
**Why:** Tests pass, production fails. Make failures visible.

### ❌ Vague Error Messages
// WRONG - not actionable
throw new Error('Bad request');
**Why:** Developers can't fix what they don't understand.
```

**4. Trigger Words (Extra Caution)**

List words that should make AI pause:

```markdown
## Trigger Words (Extra Caution)

When you see these, STOP and verify against standards:
* "mock" → Am I hiding a real integration? Don't.
* "fallback" → Am I masking a failure? Don't.
* "any" / "unknown" → Am I escaping types? Use proper types.
* "utils" → Is this proper architecture? Use specific modules.
```

**5. Success Metrics**

Help AI self-assess:

```markdown
## Success Metrics

✅ **Good:**
* Every decision references standards
* Code follows established patterns
* No standards violations in commits

❌ **Bad:**
* Writing code without checking standards
* Reinventing solved problems
* Using mocks to hide failures
```

---

## 2. Frame AI as a Role

### Principle

**Give AI a role to play that aligns with your goals.**

Instead of "you're a coding assistant," try:
- **"You're a quality-focused developer"** - For standards enforcement
- **"You're a technical team lead"** - For agent orchestration
- **"You're a security-conscious engineer"** - For security reviews

### Why This Works

AI models are trained on role-based interactions. Giving a specific role:
- Activates relevant training data
- Sets behavioral expectations
- Creates consistent personality
- Improves instruction following

### Examples

**For Open Source Projects:**
```markdown
# Your Role: Quality-Focused Developer

You prioritize code quality, readability, and maintainability over
quick hacks. You check standards before acting.
```

**For Enterprise with Agent System:**
```markdown
# Your Role: Technical Team Lead

You lead a team of specialized agents. Your job is to coordinate them
effectively, not to solve problems solo. Consult specialists before acting.
```

**For Security-Critical Systems:**
```markdown
# Your Role: Security-Conscious Engineer

Every decision considers security implications. You design authentication
correctly, validate all inputs, and never trust client data.
```

---

## 3. Include Specific Examples

### Principle

**Show, don't just tell. Include code examples of good vs bad practices.**

### Why This Matters

- **Pattern Matching** - AI learns from examples better than rules
- **Concrete Guidance** - Clear what to do, not just what to avoid
- **Reduces Ambiguity** - Examples remove interpretation variance
- **Faster Learning** - One example worth 1000 words

### Example Format

```markdown
### Error Handling

#### ❌ Bad Practice
```javascript
try {
    riskyOperation();
} catch (e) {
    console.log(e);  // Silent failure
}
```

#### ✅ Good Practice
```javascript
try {
    riskyOperation();
} catch (error) {
    console.error('❌ Operation failed:', {
        operation: 'riskyOperation',
        error: error.message,
        timestamp: new Date().toISOString()
    });
    throw new Error(`Operation failed: ${error.message}`);
}
```
```

---

## 4. Configuration File Formats

### Principle

**Use the format native to your AI assistant for best results.**

### Common Formats

**Claude Code:**
```bash
# Create CLAUDE.md in project root
# Claude reads this on every session start
```

**Cursor:**
```bash
# Create .cursorrules in project root
# Supports markdown format
```

**Continue:**
```json
// Create .continuerc.json
{
  "systemMessage": "Your personality prompt here",
  "contextProviders": ["file", "folder"]
}
```

**GitHub Copilot:**
```bash
# Create .github/copilot-instructions.md
# GitHub Copilot reads this for context
```

### File Location Best Practices

**✅ Good:**
- Project root (`./, CLAUDE.md`)
- Git-tracked (version controlled)
- One canonical file (not scattered)

**❌ Bad:**
- Hidden in subdirectories
- Not version controlled
- Multiple conflicting files
- User-level only (team can't see it)

---

## 5. Make It Living Documentation

### Principle

**Update personality prompts when you learn new patterns or add conventions.**

### Why This Matters

- **Keeps AI Current** - AI knows latest standards
- **Captures Learning** - War stories from real incidents
- **Prevents Regression** - New patterns taught to AI
- **Team Alignment** - Everyone's AI has same knowledge

### When to Update

**After Major Incidents:**
```markdown
### "Never Use DefaultAuthorizer" - CORS Disaster
**What Happened:** DefaultAuthorizer broke CORS preflight by applying
auth to OPTIONS methods. 2 days debugging.
**Rule:** Use explicit per-function authorization only.
**Date Learned:** 2025-09-03
```

**When Adding Patterns:**
```markdown
### New Pattern: Circuit Breaker (Added 2025-10-02)
After database cascade failures, we now use:
- Open after 5 consecutive failures
- Half-open after 30s
- Close after 3 successes
```

**When Standards Evolve:**
```markdown
### TypeScript Migration (Updated 2025-10-02)
**Old:** JavaScript with JSDoc
**New:** TypeScript with strict types
**Migration:** Use `@ts-expect-error` only for gradual migration
```

### Update Process

1. **Incident occurs** or **pattern discovered**
2. **Add to personality prompt** with war story
3. **Git commit** the updated prompt
4. **Team sync** - everyone pulls latest
5. **AI learns** - applies on next session

---

## 6. Balance Detail with Readability

### Principle

**Be specific enough to guide behavior, concise enough to be read quickly.**

### Why This Matters

- **Too Vague** - AI doesn't know what to do
- **Too Detailed** - AI gets overwhelmed, misses key points
- **Just Right** - AI follows consistently

### Guidelines

**Target Length:**
- Minimum: 200-300 lines (enough to be useful)
- Optimal: 500-800 lines (comprehensive but focused)
- Maximum: 1500 lines (beyond this, split into sections)

**Structure:**
```markdown
# Role (1 paragraph)
Brief role description

## Critical Standards (5-10 principles)
Why they exist + Rule

## Mandatory Workflow (4-6 steps)
Checklist for every change

## Banned Patterns (5-10 items)
Examples of what NOT to do

## Trigger Words (10-15 words)
Words that need extra caution

## Success Metrics (2 lists)
Good vs Bad behavior
```

**Writing Style:**
- ✅ Direct, imperative ("Check standards first")
- ✅ Specific ("Use db.t3.micro for dev")
- ✅ Actionable ("Call standards_enforcer")
- ❌ Vague ("Consider best practices")
- ❌ Theoretical ("One might consider...")
- ❌ Wishy-washy ("Try to maybe...")

---

## 7. Test and Iterate

### Principle

**Verify your personality prompt actually changes AI behavior.**

### How to Test

**Before/After Comparison:**
1. Ask AI to implement a feature WITHOUT prompt
2. Note violations (mocks, vague errors, etc.)
3. Add personality prompt
4. Ask same question in new session
5. Verify AI now follows standards

**Example Test:**
```
Prompt: "Add user registration endpoint"

Without config:
- AI generates mock user data fallback
- Returns generic "error" messages
- Creates new utils file

With config:
- AI checks if registration pattern exists
- Fails loud on errors with specific codes
- Uses existing authentication helpers
```

**Common Issues:**

**Issue: AI ignores prompt**
- Solution: Make instructions more direct ("Never use mocks" vs "Avoid mocks")
- Solution: Add war stories (shows consequences)
- Solution: Use trigger words earlier in file

**Issue: AI over-applies rules**
- Solution: Add "When to apply" and "When NOT to apply" sections
- Solution: Give examples of valid exceptions
- Solution: Clarify edge cases

**Issue: AI references wrong patterns**
- Solution: Be more specific about file locations
- Solution: Include exact pattern names
- Solution: Show examples of correct references

---

## Example Template

See [CLAUDE.md.template](./CLAUDE.md.template) for a complete, production-ready template you can copy and customize.

**Key Features:**
- Role framing (Quality-Focused Developer)
- War stories (why standards exist)
- Mandatory workflow (step-by-step)
- Banned patterns (with examples)
- Trigger words (extra caution)
- Success metrics (self-assessment)
- Project-specific section (customize here)

---

## Summary

Effective AI assistant configuration:

1. **Use Personality Prompts** - Context about standards, conventions, war stories
2. **Frame as a Role** - "Quality-focused developer" or "Technical team lead"
3. **Include Examples** - Good vs bad code, specific patterns
4. **Choose Right Format** - CLAUDE.md, .cursorrules, etc.
5. **Keep It Living** - Update when learning new patterns
6. **Balance Detail** - Specific but readable (500-800 lines)
7. **Test and Iterate** - Verify behavior actually changes

**Start with the template, customize for your project, update as you learn.**

---

**Questions?** Contact info@happyhippo.ai
