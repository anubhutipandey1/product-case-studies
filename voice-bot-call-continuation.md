# Feature Case Study: Voice Bot Call Continuation

**Context:** Product Manager on a conversational AI platform serving voice bots at scale across multiple business verticals. This case study documents a feature I specced, defined success criteria for, and shipped to production.

---

## The Problem

When a user's call dropped mid-conversation with a voice bot, the system had no memory of the interaction. On reconnecting, the user was greeted with the standard welcome message and forced to start over — repeating their intent, re-navigating menus, and re-providing context they had already given.

This created two compounding frustrations:
- **User effort:** the user had to redo work they had already done
- **Broken trust:** the bot appeared to have "forgotten" them entirely, undermining confidence in the system

For high-volume voice bots handling service requests, complaints, or transactions, call drops are not edge cases — they happen regularly due to network conditions, especially in low-connectivity environments. The status quo was silently eroding user experience at scale.

---

## Users Affected

**Primary:** End users calling into automated voice bot services — particularly in contexts where calls are long, multi-step, or involve sensitive information (account queries, service requests, support escalations).

**Secondary:** Business teams deploying voice bots — they were absorbing the reputational cost of a poor re-entry experience without a lever to fix it.

---

## Feature Design

### Core concept
When a returning caller is detected, instead of the standard welcome flow, the bot offers them a choice: *resume where you left off* or *start a new conversation.* The user stays in control — nothing is assumed.

### What I chose not to do
An earlier consideration was to *automatically* resume the previous conversation without asking. I rejected this because:
- Users may have resolved their issue and be calling for a different reason
- Automatically resuming could feel intrusive — the bot "knowing too much"
- A yes/no prompt costs three seconds and gives the user agency

This is a deliberate UX tradeoff: slightly more friction, significantly more trust.

### Configuration-first design
The feature is not enabled by default. Bot operators must explicitly turn it on in configuration. This was an intentional decision for three reasons:

1. **Not all bots benefit from it** — a simple FAQ bot with 30-second interactions doesn't need call continuation; a complex service request bot does
2. **Operators need to configure the "resume" intent** — the feature requires a mapped intent for the restored conversation state, which takes setup effort
3. **It prevents unintended behaviour** in bots where operators haven't thought through the resumed flow

### Conversation flow

```
User calls in
     │
     ▼
Is Call Continuation enabled for this bot?
     │
   No ──────────────────────────► Standard welcome message
     │
   Yes
     │
     ▼
API call: Is this a repeat caller? (lookup by mobile number)
     │
   No ──────────────────────────► Standard welcome message
     │
   Yes
     │
     ▼
Play dynamic message:
"Welcome back. Would you like to continue
 where we left off, or start a new conversation?"
     │
     ├── User says YES ──────────► Restore last intent → resume conversation
     │
     ├── User says NO ───────────► Standard welcome intent → new conversation
     │
     └── Garbage / no response ──► Fallback handler → standard welcome intent
```

### Key technical dependencies
- **Repeat caller API:** identifies returning callers by mobile number and retrieves their last conversation state
- **Intent restoration:** the bot's last active intent is stored and passed back into the conversation engine on resume
- **Bot configuration flag:** a per-bot toggle (`call_continuation: enabled/disabled`) with a mandatory intent mapping field when enabled — the platform enforces this at configuration time to prevent misconfigured deployments

---

## Tradeoffs Considered

| Decision | Alternative considered | Why I chose this approach |
|---|---|---|
| Ask the user before resuming | Auto-resume silently | Preserves user agency; avoids assuming context |
| Opt-in per bot (not default on) | Enable for all bots by default | Not universally applicable; prevents misconfigured rollouts |
| Mandatory intent configuration when enabled | Make intent config optional | Optional config would create silent failures — bots enabled but with no resume path |
| Fallback to welcome intent on garbage input | Retry the resume prompt | Retry loops frustrate users; a clean fallback is safer |

---

## Success Metrics

I defined success at two levels: **adoption** (are operators using this?) and **effectiveness** (is it working for users?).

### Adoption
- Number of bots with call continuation enabled — indicates operator confidence in the feature

### Effectiveness (per bot, per period)
- **Resume rate:** percentage of repeat callers who choose to continue the previous conversation — the primary signal that the feature is valuable
- **Decline rate:** percentage of repeat callers who choose to start a new conversation — healthy to have; indicates users calling for a different reason
- **Garbage input rate on the resume prompt:** high rates indicate the prompt phrasing is unclear or the speech recognition is mishandling responses
- **Intent restoration success rate:** percentage of resumed conversations where the last intent is successfully triggered — a technical health metric
- **Garbage input rate post-restoration:** if users are confused after the intent is restored, the resumed state may not be contextually clear enough

### What these metrics tell you together
A healthy deployment looks like: high resume rate, low garbage input rates at both decision points, near-100% intent restoration success. If garbage input on the resume prompt is high, the message needs rewording. If garbage input post-restoration is high, the intent state being restored is too ambiguous — the bot may need to re-surface context before continuing.

---

## What I'd Do Differently

**Add a session expiry threshold.** The current design doesn't define how old a "previous conversation" can be before it's no longer worth resuming. A call from 3 minutes ago is obviously resumable. A call from 6 days ago probably isn't — the user's situation has likely changed. I'd add a configurable expiry window (e.g. resume only if the last call was within 24 hours) to prevent the bot from surfacing stale context.

**Track resume-to-resolution rate.** The current success metrics measure whether users *chose* to resume and whether the intent *fired* — but not whether the resumed conversation actually *resolved the user's issue*. The most meaningful signal would be: of users who resumed, what percentage completed their original task? This requires connecting conversation state to outcome data, which is a larger instrumentation effort but worth prioritising in a future iteration.

**A/B test the prompt phrasing.** The resume prompt is a single fixed message. Different phrasings — more explicit ("Continue your previous call about X"), more neutral ("Resume where you left off"), more concise ("Continue previous call?") — will likely produce meaningfully different resume rates. I'd run a controlled test across a high-volume bot to find the optimal phrasing before rolling it out more broadly.

---

## PM Reflection

This feature is a good example of a class of problems I find genuinely interesting: **state recovery in interrupted flows.** The technical mechanism (store intent, restore on repeat call) is straightforward. The harder design questions are entirely product: how long is context valid? who decides? what does the user control? how do you measure whether it worked?

The success criteria I defined for this feature — particularly the garbage input rates at multiple points in the flow — reflect a conviction I hold about metrics: you should measure *where users struggle*, not just *whether they succeeded*. The difference between "resume rate" and "garbage input rate on the resume prompt" is the difference between knowing something worked and understanding *why* it worked or didn't.

---

*This case study is a sanitised version of a feature I designed and delivered on a production voice bot platform. Internal system names, configuration UIs, and proprietary architecture details have been removed.*
