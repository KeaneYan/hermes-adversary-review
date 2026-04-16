---
name: adversary-review
description: Use a delegated subagent to adversarially review drafts before sending them, reducing factual mistakes, overclaiming, and omissions.
version: 1.1.0
---

# Adversary Review

Use a delegated subagent to adversarially review important drafts before sending them to the user or posting them externally.

This skill helps catch:
- factual mistakes
- overclaiming
- missing caveats
- weak or risky advice
- tone problems
- unnecessary verbosity

## When to Use

Prioritize **risk**, not just length.

### Quick Decision Table

| Situation | Default action |
|-----------|----------------|
| Simple greeting / confirmation / raw tool output | Skip review |
| Short technical reply with factual judgment | Use **quick review** |
| Long reply with technical advice, root-cause claims, or decisions | Use **full review** |
| External-facing draft (issue / PR / email / announcement) | Review by default |
| Sensitive content with secrets / PII / credentials | Skip review |
| Need structured output for later automation | Use **JSON review** |

### One-line rule
- **Low risk** → skip
- **Medium risk** → quick review
- **High risk / external** → full review
- **Need structured output** → JSON review

## How to Review

Use `delegate_task` with **no tools**.

Recommended defaults:
- `toolsets: []`
- `max_iterations: 2` or `3`
- Send only the draft and the minimum required context

```python
delegate_task({
  goal: "Review the draft below and identify problems before it is sent",
  context: "<review prompt + draft>",
  toolsets: [],
  max_iterations: 2
})
```

If the draft is long, send only:
- the final text you plan to send
- the critical context constraint
- the one or two review priorities you care most about

## Review Modes

### Mode A — Quick Review

Use for:
- short technical updates
- issue / PR / commit summaries
- short external English drafts
- quick “is this safe to send?” checks

Prompt:

```text
You are a strict but concise reviewer. Check the draft below for:
1. factual mistakes
2. overclaiming
3. important omissions
4. tone problems

[Draft]
"""
{draft}
"""

Reply in exactly one of these formats:
- PASS: <one sentence>
- ISSUES:
  - <issue 1>
  - <issue 2>
```

### Mode B — Full Review

Use for:
- technical conclusions
- long replies
- multi-step recommendations
- external-facing formal drafts
- important root-cause analyses

Prompt:

```text
You are an adversarial reviewer. Strictly review the draft below.

[Draft]
"""
{draft}
"""

Check for:
1. factual mistakes, unverified claims, overconfidence
2. missing caveats, boundaries, or risk notes
3. logical contradictions or broken causal chains
4. tone problems (glib, apologetic, overly certain, performative)
5. advice that could mislead the user or cause problems
6. repetition or avoidable verbosity
7. one high-value missing sentence worth adding

Reply in exactly one of these formats:
- PASS: <one-sentence verdict>
- ISSUES:
  - location
  - problem
  - why it is a problem
  - suggested fix

Rules:
- Be strict
- Do not rewrite the whole draft
- Do not output filler
- If something is weak but not wrong, treat it as a suggestion, not a false error
```

### Mode C — JSON Review

Use when the result needs to be easy to parse or act on automatically.

Recommended JSON shape:

```json
{
  "pass": true,
  "summary": "One-line verdict",
  "issues": []
}
```

If there are issues:

```json
{
  "pass": false,
  "summary": "Two issues need revision",
  "issues": [
    {
      "severity": "high",
      "location": "paragraph 2, sentence 1",
      "problem": "The claim is stated as confirmed without evidence.",
      "why": "This overstates certainty and may mislead the reader.",
      "suggestion": "Use a more qualified statement."
    }
  ]
}
```

JSON prompt:

```text
You are an adversarial reviewer. Review the draft below.

[Draft]
"""
{draft}
"""

Check for:
1. factual mistakes
2. overclaiming
3. important omissions
4. insufficient caveats
5. tone problems

Output valid JSON only. No markdown. No explanation outside JSON.

JSON schema:
{
  "pass": true,
  "summary": "One-line verdict",
  "issues": [
    {
      "severity": "low|medium|high",
      "location": "location",
      "problem": "problem",
      "why": "why",
      "suggestion": "suggestion"
    }
  ]
}

Rules:
- If there are no issues, set `pass=true` and `issues=[]`
- If there is any clear issue, set `pass=false`
- Output must be parseable JSON
```

## Handling Review Results

- `PASS:` → send the draft
- `ISSUES:` → fix only the valid issues, then send the revised version
- Bad review / obvious false positives → ignore what is not useful

Important:
- The reviewer is a guardrail, not the final authority
- Do not blindly apply every suggestion
- If the review causes real changes, just send the improved draft; no need to mention the review step

## Real Examples

### Example 1 — Quick Review

Draft:

```text
Root cause confirmed. The April auxiliary routing change made GPT-5 named custom providers fall back to chat_completions, which breaks aixj.vip and causes 502s.
```

Use this when you want a fast “safe to send?” pass.

### Example 2 — Full Review

Draft:

```text
I traced the failure to auxiliary runtime resolution. The main agent upgrades GPT-5 models to codex_responses, but named custom providers in runtime_provider.py did not mirror that behavior when api_mode was omitted. As a result, auxiliary tasks could hit chat_completions and fail on relays that only work through /v1/responses. I fixed it by adding model-based API-mode upgrading for both the normal named-custom path and the credential-pool path.
```

Use this when you need scrutiny on certainty, boundaries, and technical accuracy.

### Example 3 — JSON Review

Draft:

```text
PR is ready to merge. All relevant tests passed and no unrelated files were included.
```

Possible result:

```json
{
  "pass": false,
  "summary": "One claim should be softened.",
  "issues": [
    {
      "severity": "medium",
      "location": "sentence 1",
      "problem": "'ready to merge' is too strong if CI or review is still pending.",
      "why": "It can overstate completion.",
      "suggestion": "Use 'ready for review' or another more conservative phrase."
    }
  ]
}
```

## Model Selection

- The review model comes from Hermes `delegation.provider` / `delegation.model`
- If delegation is not configured, the child inherits the parent model/provider
- Do **not** add unsupported `model: {...}` fields to `delegate_task(...)`
- `delegate_task` supports `goal`, `context`, `toolsets`, `tasks`, `max_iterations`, `acp_command`, and `acp_args`

## Cost / Latency

- Usually adds about 2–5 seconds
- Usually costs a few hundred tokens
- Skip low-risk replies to save time and cost
- For long drafts, trim context aggressively

## Notes

- Send only the draft and required constraints, not the full conversation history
- Skip review if the draft contains secrets, credentials, or PII
- The child reviewer usually needs no tools at all
