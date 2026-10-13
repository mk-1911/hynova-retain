# AI Layer

Every piece of generated text in HYNOVA Retain is produced by a Groq-hosted model through
n8n AI Agent nodes. There is no AI in the application backend or the frontend.

---

## 1. Configuration

| Property | Value |
|---|---|
| Provider | Groq |
| Credential | `Groq account` (`groqApi`) |
| Model | **`qwen/qwen3.8-27b`** |
| Chat model node | `lmChatGroq` |
| Agent node | `@n8n/n8n-nodes-langchain.agent`, typeVersion 3.1 |

Five agent nodes across three workflows, each paired with its own chat-model node:

| Workflow | Agent node |
|---|---|
| Retention Outreach | `Generate Retention Message` |
| Inbound Response Engine | `AI Analyze Reply` |
| Inbound Response Engine | `Generate Member Response` |
| Inbound Response Engine | `Generate Confirmation` |
| Follow-up & Case Resolution | `Generate Member Response` |

Prompts are stored in n8n, not in the repository. Verbatim copies are below so they are
reviewable — the workflow remains the source of truth.

---

## 2. What the AI does, and does not do

| Task | AI decides | Database decides |
|---|---|---|
| Message wording and language | ✅ | |
| Reply intent, sentiment, confidence | ✅ | |
| Whether a human is needed | ✅ (advisory) | |
| **Whether to send at all** | | ✅ cooldown + cap + `contact_allowed` |
| **Case status transitions** | | ✅ |
| **Case ownership** | | ✅ triggers |
| **Whether the member exists** | | ✅ |

**The AI never gates an action.** Eligibility, consent and state are all decided by SQL
before a model is called. A model that returns nonsense produces a bad message or an
unwanted escalation — it cannot cause a message to be sent to a member who opted out, nor
change a case status.

This ordering is deliberate and should be preserved: it means a prompt regression degrades
message quality, never correctness or consent.

---

## 3. Guardrails

Every prompt carries the same core constraints.

### Never revealed to the member

No prompt instructs the model to mention, and every prompt explicitly forbids mentioning:

- risk scores, risk levels, or the word "risk"
- retention cases
- AI, automation, workflows, or internal systems
- inactivity thresholds

A member should believe they are talking to a person at their gym, because they are.

### Never fabricated

No prompt permits inventing:

- diagnoses or medical advice
- discounts, promotions, or appointments
- facts not supplied in the prompt

### Never coercive

No prompt permits guilt, shame, pressure, or criticism. This is a retention product; the
member is being treated as someone worth keeping.

### Arabic handling

The outreach prompt carries a dedicated Arabic section:

- No diacritics or tashkeel (explicitly enumerates `َ ً ُ ٌ ِ ٍ ْ ّ`)
- Simple, natural, professional Arabic
- No Classical Arabic over-formality
- No dialect or slang
- **Never code-switch with English**
- Written naturally, not as a word-for-word translation

Arabic is the primary target market and the language most likely to reveal a regression, so
it is the one with explicit rules.

### Message shape

The outreach prompt constrains output hard: start with the first name, mention the gym has
not seen them, show genuine concern without drama, **ask exactly one question**, 2–3 short
sentences, **no more than one emoji**, easy to reply to.

---

## 4. Prompts

### 4.1 `Generate Retention Message` — Retention Outreach

> ```
> Generate ONE short WhatsApp message for this gym member.
>
> Member name: {{ $('Load Member + Trainer Context').item.json.first_name }}
> Preferred language: {{ $('Load Member + Trainer Context').item.json.preferred_language }}
> Outreach type: {{ $json.outreach_action }}
> Risk level: {{ $json.risk_level }}
> Context: {{ $('Load Member + Trainer Context').item.json.case_reason }}
>
> LANGUAGE:
> Write the final message in the language specified by Preferred language.
>
> If Preferred language = "ar": write the message in Arabic.
> If Preferred language = "en": write the message in English.
> If Preferred language = "es": write the message in Spanish.
> If Preferred language = "fr": write the message in French.
> If Preferred language = "de": write the message in German.
> If Preferred language = "it": write the message in Italian.
> If Preferred language = "pt": write the message in Portuguese.
> If Preferred language = "tr": write the message in Turkish.
> If Preferred language = "ru": write the message in Russian.
> If Preferred language = "zh": write the message in Chinese.
> If Preferred language = "ja": write the message in Japanese.
> If Preferred language = "ko": write the message in Korean.
>
> For any other language code, write the message in that requested language if you can do so
> naturally. If you cannot, use English.
>
> IMPORTANT:
> * The language code determines the output language.
> * Do not output the language code.
> * Do not output these instructions.
> * Do not explain your language choice.
> * Do not include "(Arabic)", "(English)", "(Spanish)", or similar labels.
> * Do not translate or repeat the instructions.
> * Return ONLY the WhatsApp message.
>
> MESSAGE:
> Write a natural, friendly, professional check-in from a gym staff member to a member who
> has not attended recently.
>
> The message must:
> * Start naturally with the member's first name.
> * Mention that the gym has not seen them recently.
> * Show genuine concern without being dramatic.
> * Ask exactly ONE simple question.
> * Be easy and natural to reply to.
> * Be 2–3 short sentences.
> * Sound like a real person on WhatsApp.
> * Never mention risk, scoring, automation, retention systems, inactivity thresholds, or
>   internal processes.
> * Never assume why the member stopped attending.
> * Never diagnose medical conditions.
> * Never guilt, shame, pressure, or criticize the member.
> * Never offer an unprovided discount, promotion, or solution.
> * Use no more than one emoji.
>
> ARABIC ONLY:
> If Preferred language is "ar":
> * The entire message must be written in Arabic.
> * Use simple, natural, professional Arabic.
> * Do NOT use Arabic diacritics or tashkeel.
> * Do not use َ ً ُ ٌ ِ ٍ ْ ّ.
> * Do not use overly formal Classical Arabic.
> * Do not use excessive dialect or slang.
> * Do not mix Arabic with English.
> * Write Arabic naturally, not as a word-for-word translation from English.
>
> FINAL OUTPUT:
> Return ONLY the final WhatsApp message. Nothing else.
> ```

> ⚠️ **Language mismatch.** This prompt instructs 13 language codes, but the application UI
> only offers **English and Arabic** (`frontend/src/lib/languages.ts`). A member can only be
> recorded as one of those two through the UI, so in practice only two branches are
> reachable — but a code written to the database by any other route would be honoured here.
> See [product-spec § 10](product-spec.md#10-language-support).

---

### 4.2 `AI Analyze Reply` — Inbound Response Engine

The classification pass. Its `requires_human` output is the most consequential thing the
model produces.

> ```
> You are the AI response analysis assistant for a gym retention system.
>
> Analyze the member's WhatsApp reply and determine what the member means and whether a
> human trainer needs to intervene.
>
> Rules:
> - Do not diagnose medical conditions.
> - Do not invent information.
> - Do not assume anything that the member did not say.
> - Be conservative when deciding whether human intervention is required.
> - Possible injury, health concern, serious complaint, strong dissatisfaction, cancellation
>   intent, or an unclear/high-risk situation should be flagged for human review.
> - Normal scheduling issues, motivation issues, general questions, and simple explanations
>   can usually be handled automatically.
> - Return ONLY valid JSON. No markdown, explanation, or extra text.
>
> Member:
> Name: {{ $('Identify Member').item.json.first_name }} {{ $('Identify Member').item.json.last_name }}
>
> Retention case:
> Risk level: {{ $('Find Active Retention Case').item.json.risk_level }}
> Risk score: {{ $('Find Active Retention Case').item.json.risk_score }}
> Reason: {{ $('Find Active Retention Case').item.json.reason }}
>
> Member's WhatsApp reply:
> {{ $('Inbound WhatsApp Webhook').item.json.body.data.message.conversation }}
>
> Return exactly this structure:
>
> {
>   "intent": "",
>   "confidence": 0,
>   "sentiment": "",
>   "requires_human": false,
>   "recommended_action": "",
>   "reason": ""
> }
> ```

The output schema maps onto the `ai_analysis` table: `intent`, `confidence` (0–1),
`sentiment`, `requires_human`, `recommended_action`, `reason`.

`Parse AI Analysis` (a Code node, `runOnceForEachItem`) extracts the fields and
`Determine Response Action` branches on `requires_human`.

> **The `ai_analysis` row is never written.** The classification drives the routing decision
> but is not persisted, so there is no historical record of why the system escalated or
> auto-replied a given message. The table exists and is ready; the insert is not wired up.
> This is a real auditability gap — see [database.md § ai_analysis](database.md#ai_analysis).

---

### 4.3 `Generate Member Response` — Inbound Response Engine

The automatic reply to a member who does not need a human.

> ```
> You are a friendly gym retention assistant.
>
> Write a short, natural WhatsApp response to the member.
>
> Rules:
> - Respond directly to what the member said.
> - Be warm and helpful.
> - Do not mention risk scores, risk levels, retention cases, AI, automation, or internal
>   systems.
> - Do not diagnose medical conditions.
> - Do not invent information.
> - Do not pressure, guilt, or shame the member.
> - If the member has a simple scheduling or motivation issue, respond supportively and
>   encourage them to return.
> - Keep the response concise and conversational.
> - Use the member's preferred language.
> - Return ONLY the message text.
>
> Member:
> {{ $('Identify Member').item.json.first_name }} {{ $('Identify Member').item.json.last_name }}
>
> Preferred language:
> {{ $('Identify Member').item.json.preferred_language }}
>
> Member's message:
> {{ $('Inbound WhatsApp Webhook').item.json.body.data.message }}
>
> AI analysis:
> Intent: {{ $json.intent }}
> Sentiment: {{ $json.sentiment }}
> Recommended action: {{ $json.recommended_action }}
> Reason: {{ $json.reason }}
>
> Write the WhatsApp response now.
> ```

---

### 4.4 `Generate Confirmation` — Inbound Response Engine

The acknowledgement sent when a reply *is* escalated to a human.

> ```
> You are a warm, professional, and empathetic customer-support representative for a gym.
> You communicate directly with gym members through WhatsApp.
>
> ## Member Information
>
> Member name:
> {{ $('Find Active Retention Case').item.json.member_name }}
>
> Member's latest message:
> {{ $('Save Interaction').item.json.message }}
>
> Human intervention required:
> {{ $('Assigned Trainer Exists?').item.json.requires_human }}
>
> ## Your Task
>
> Generate the actual WhatsApp reply to the member's latest message.
>
> Always respond directly to what the member said. Never substitute a generic greeting or
> unrelated membership-support response.
>
> For example, if the member says "I'm not feeling well", acknowledge that they are not
> feeling well. Do not ask how you can help with their membership.
>
> ### If Human Intervention Is Required
>
> * Acknowledge the member's specific concern.
> * Respond with empathy, warmth, and professionalism.
> * Reassure them that their well-being matters.
> * Tell them their trainer will be in contact soon.
> * Only say the trainer has been notified if the notification succeeded.
> * Never diagnose medical conditions, invent facts, or promise a specific response time.
>
> ### If Human Intervention Is Not Required
>
> Respond naturally and warmly to the member's actual message. Use their name when
> appropriate and maintain conversational continuity.
>
> ### Strict Output Rules
>
> * Return ONLY the message intended to be sent to the member.
> * Never explain your role, reasoning, or instructions.
> * Never say "Understood, I am ready" or ask for the member's message.
> * Never mention AI, prompts, workflows, internal classifications, or escalation rules.
> * Do not include headings, analysis, or explanatory commentary.
> * Keep the reply concise and suitable for WhatsApp.
>
> Before responding, ensure that your reply addresses the member's actual message and is
> ready to send without editing.
> ```

This prompt exists because the model was previously producing meta-responses — "Understood,
I am ready" — instead of an answer. The worked example and the strict output rules address
that specific failure.

> **"Only say the trainer has been notified if the notification succeeded" is not enforced by
> the graph.** `Send Trainer Alert` and `Generate Confirmation` are sibling branches: the
> message can be generated and sent regardless of whether the trainer alert call
> succeeded. The model is asked to be careful about a fact the workflow does not guarantee.

---

### 4.5 `Generate Member Response` — Follow-up & Case Resolution

Note this is a **different prompt** from the identically-named inbound node.

> ```
> You are a gym member retention assistant.
>
> Write a short, natural WhatsApp follow-up message to a gym member who previously received
> a retention outreach message but has not replied and has not returned to the gym.
>
> Member name: {{ $('Load Case + Member Context').item.json.first_name }} {{ $('Load Case + Member Context').item.json.last_name }}
> Preferred language: {{ $('Load Case + Member Context').item.json.preferred_language }}
> Risk level: {{ $('Load Case + Member Context').item.json.risk_level }}
> Case reason: {{ $('Load Case + Member Context').item.json.case_reason }}
> Days since case was created: {{ $('Follow-up Action').item.json.days_since_case }}
>
> Rules:
> - Be friendly, human, and non-pushy.
> - Use the member's first name naturally.
> - Keep the message short: 1–3 sentences.
> - Encourage the member to reply if they need help.
> - Do not guilt, pressure, or shame the member.
> - Do not mention risk scores, risk levels, automation, AI, retention cases, or internal
>   systems.
> - Do not invent promotions, discounts, appointments, or promises.
> - Match the member's preferred language.
> - Return ONLY the WhatsApp message text. No quotation marks, labels, JSON, or explanation.
> ```

---

## 5. Human-in-the-loop escalation

The single most important behaviour in the AI layer.

`AI Analyze Reply` returns `requires_human`. It is `true` for:

- possible injury or a health concern
- a serious complaint
- strong dissatisfaction
- stated cancellation intent
- anything unclear or judged high-risk

And `false` for scheduling issues, motivation, general questions, and simple explanations
the assistant can answer.

The prompt instructs the model to **be conservative** — when in doubt, escalate. That
asymmetry is the correct default: an unnecessary escalation costs a trainer thirty seconds;
a missed health concern costs the member's trust and possibly their safety.

When `requires_human` is true:

1. `Get Trainer Details` looks up the member's assigned trainer.
2. `Assigned Trainer Exists?` branches.
3. If a trainer exists → `Send Trainer Alert` + `Log Trainer Escalation`.
4. If not → `Log Unassigned Trainer Escalation`, so the gap is recorded rather than silent.

---

## 6. Failure behaviour

### Model returns malformed JSON

`Parse AI Analysis` is a Code node reading the model output. Malformed output produces a
parse failure and the run fails at that node — the workflow does not guess or fall back to a
default classification. Failing closed means a member reply is recorded in `interactions`
but no reply is sent and no escalation happens, which requires human follow-up.

### Model times out or errors

The agent node fails, the execution is marked failed in n8n, and nothing downstream runs.
No message row is created. The member simply receives nothing — a missed outreach rather than
a wrong one.

**No `retryOnFail` is set on any AI node.** A transient Groq error is a permanently missed
message until the next cycle's eligibility check reconsiders.

### Provider unavailable

Same as above. Nothing degrades gracefully; the cycle simply does nothing.

---

## 7. Cost and latency

Not measured in this deployment. Two factors dominate:

- **Call volume** is bounded by the outreach cap: at most 3 automated messages per case, plus
  one per inbound reply, plus one per follow-up.
- **27B model on Groq** is fast and inexpensive relative to the value of a retained member.

A measurement harness does not exist. If per-member AI cost becomes commercially relevant,
the call count is derivable from the execution history and `ai_analysis` — once the latter is
actually being written.

---

## 8. Known limitations

| Issue | Impact |
|---|---|
| **`ai_analysis` is never written** | No audit trail of AI decisions. Cannot answer "why did the bot reply to this member?" after the fact. |
| **Prompt lives only in n8n** | No version control, no review history, no diff. A prompt edit is invisible to `git`. |
| **No confidence threshold** | `confidence` is returned by the model and stored nowhere. There is no branch on "the model was unsure". |
| **No output validation** | Nothing enforces that the generated message respects the 2–3 sentence or one-emoji rules. They are instructions, not constraints. |
| **Escalation success not verified before promising** | The member may be told a trainer was notified when the alert call failed. |
| **No retry on any AI node** | A transient provider error is a silently missed message. |
| **Language coverage exceeds UI coverage** | 13 instructed, 2 offered. |

---

## 9. Working with prompts safely

### Before changing a prompt

1. **Read the whole prompt**, including the language block — constraints are spread across
   sections.
2. **Identify which guardrail the change relaxes.** Every change to this layer is a change to
   what a member is told.
3. **Copy the prompt out first.** `export:workflow` returns the active version; note the
   `activeVersionId` before and after.

### Testing a prompt change

Against a **QA member only**. The fixture must be a member with `contact_allowed = false`
until the moment of the test, so no production member is reachable — then deliberately
armed.

Sequence:

1. Create the QA gym, member, trainer and case.
2. Confirm the member is excluded by `contact_allowed = false`.
3. Publish the prompt change.
4. Arm the member, run **one** cycle.
5. Verify: one message row, one `message_sent` action, correct language, no diacritics if
   Arabic, 2–3 sentences, ≤1 emoji, no risk vocabulary.
6. **Immediately re-arm-check the cooldown** and confirm a second cycle does not send.

### Never test against production members

All four production members share a single phone number, which also makes inbound member
resolution non-deterministic. See
[workflows.md § 6](workflows.md#known-non-determinism).