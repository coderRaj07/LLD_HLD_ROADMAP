# Designing WhatsApp Schema Like a Senior Engineer

## From Wrong Schema → Right Schema Using Abstractions, Relationships & Query Thinking

Most developers start schema design by thinking:

> “What tables should I create?”

Senior engineers think differently.

They ask:

```text
1. What is the core thing?
2. What exists independently?
3. What connects things?
4. What happens over time?
5. Where does state belong?
6. Am I creating branching chaos?
```

This single framework completely changes how you design systems.

Let’s design a WhatsApp-like messaging system step by step and understand:

* why naive schemas fail
* how abstractions emerge
* what a junction table REALLY is
* why messages are NOT junction tables
* how unread logic naturally emerges from good modeling

---

# Problem Statement

We want to support:

* 1-1 chat
* group chat
* notifications
* unread counts
* online/offline
* media
* reactions
* pagination
* multi-device
* future channels/community support

---

# Step 1 — What Is the Core Thing?

Most beginners think:

```text
personal_chat
group_chat
```

This immediately creates:

```python
if personal_chat:
elif group_chat:
```

everywhere.

This is called:

# IF-ELSE HELL

A senior engineer asks:

> What is the user REALLY interacting with?

Answer:

# Conversation

Everything revolves around conversation.

Because:

* personal chat = conversation
* group chat = conversation
* channel = conversation
* support inbox = conversation

So our stable abstraction becomes:

## conversation

```sql
conversation
------------
id
name
type
created_at
created_by
```

---

# Step 2 — What Exists Independently?

Now identify first-class business objects.

Rule:

> Can product/business talk about this as a real thing?

Examples:

* delete message
* search message
* pin conversation

These are real objects.

So our independent entities become:

## user

```sql
user
----
id
name
mobile
```

---

## conversation

```sql
conversation
------------
id
name
type
created_at
created_by
```

---

## message

```sql
message
-------
id
conversation_id
sender_id
content
created_at
```

---

# Important Confusion — “Message Has Foreign Keys, So Is It Junction Table?”

This is one of the biggest beginner confusions.

Because message contains:

```sql
conversation_id
sender_id
```

people think:

> “Is message also a junction table?”

NO.

And understanding WHY is extremely important.

---

# What Is a Junction Table REALLY?

A junction table’s PRIMARY purpose is:

# connecting entities

Example:

## conversation_participant

```sql
conversation_participant
------------------------
user_id
conversation_id
```

Meaning:

```text
User belongs to conversation
```

The row itself IS the relationship.

Without:

* user
* conversation

the row has no meaning.

---

# Message Is Different

## message

```sql
message
-------
id
conversation_id
sender_id
content
created_at
```

This row represents:

```text
“Raj sent hello in conversation 100”
```

That is:

* a real business object
* an event
* searchable
* editable
* deletable
* replyable
* reactable

So message is:

# an entity/event table

NOT a junction table.

---

# The Real Difference

## Junction Table

Represents:

```text
connection/state between entities
```

Example:

```text
user belongs to conversation
```

---

## Entity/Event Table

Represents:

```text
something meaningful happened
```

Example:

```text
Raj sent hello
```

---

# Golden Rule

A better test is:

# “If I remove one FK, does the row lose all meaning?”

---

## conversation_participant

Remove:

```text
conversation_id
```

Meaning gone.

Remove:

```text
user_id
```

Meaning gone.

Because table ONLY exists to connect.

---

## message

Even though linked:

* message still conceptually exists
* it is still a communication event

So message is NOT a junction table.

---

# Step 3 — What Connects Things?

Now ask:

> Which entities are connected?

We discover:

```text
user <=> conversation
```

This becomes:

## conversation_participant

```sql
conversation_participant
------------------------
id
user_id
conversation_id
joined_at
```

---

# Why Not Store Arrays?

Bad design:

```sql
conversation
------------
participants = [1,2,3]
```

Looks simple initially.

But then future features arrive:

* mute chat
* role
* unread count
* joined_at
* pinned
* archived
* nickname

All these belong to:

```text
(user <=> conversation)
```

relationship state.

Arrays cannot model this cleanly.

So relationship becomes first-class.

---

# Step 4 — What Happens Over Time?

Now think about events.

Messages are events.

```text
Raj sent message
```

is something that happened over time.

That’s why:

```sql
created_at
```

is core to message.

This is why chat systems naturally become event-driven systems.

---

# Huge Insight — Messages DO NOT Define Membership

This is where many people get confused.

They think:

> “Can’t I derive participants from message table?”

NO.

Example:

Group has:

```text
Raj
Aman
Neha
John
```

Only Raj sends messages.

Message table:

```text
conversation_id | sender_id
--------------------------------
100             | Raj
100             | Raj
100             | Raj
```

Now ask:

# Can you derive Aman, Neha, John?

NO.

Because:

* they never sent messages
* but still belong to group
* still receive messages
* still have unread counts

This is why:

## membership ≠ messaging

---

# Final Foundational Schema

## conversation

```sql
conversation
------------
id
name
type
created_at
created_by
```

---

## user

```sql
user
----
id
name
mobile
```

---

## message

```sql
message
-------
id
conversation_id
sender_id
content
created_at
```

---

## conversation_participant

```sql
conversation_participant
------------------------
id
user_id
conversation_id
joined_at
```

---

# Step 5 — Validate Schema Using Queries

This is how senior engineers test schemas internally.

Ask:

> “Can difficult product queries be expressed cleanly?”

If queries become:

* ugly
* branchy
* full of special cases

schema is probably wrong.

---

# Query — Top 5 Messages Visible to Rajendra

Assume:

```text
user_id = 10
```

Query:

```sql
WITH user_conversations AS (
    SELECT conversation_id
    FROM conversation_participant
    WHERE user_id = 10
)

SELECT m.content, m.sender_id
FROM message m
WHERE m.conversation_id IN (
    SELECT conversation_id
    FROM user_conversations
)
ORDER BY m.created_at DESC
LIMIT 5;
```

Notice how naturally the query emerged.

---

# Step 6 — Unread Messages

Now we hit a real product requirement:

> “Find unread messages for Rajendra”

Naive engineers do:

```sql
message_read_status
-------------------
message_id
user_id
is_read
```

This explodes massively.

---

# Senior Insight

Unread is NOT:

```text
message state
```

Because:

* Raj may have read message
* Aman may not

simultaneously.

Unread belongs to:

```text
(user <=> conversation)
```

relationship state.

---

# Elegant Solution

Add:

## conversation_participant

```sql
conversation_participant
------------------------
user_id
conversation_id
last_read_message_id
```

Now unread means:

```text
all newer messages
```

Beautifully scalable.

---

# Unread Query

```sql
SELECT m.*
FROM message m
JOIN conversation_participant cp
    ON m.conversation_id = cp.conversation_id
WHERE cp.user_id = 10
AND m.id > cp.last_read_message_id
AND m.sender_id != 10
ORDER BY m.created_at DESC
LIMIT 5;
```

---

# Observe the Beauty

Unread logic naturally emerged from:

```text
(user <=> conversation)
```

relationship state.

No ugly:

```sql
message_read_status
```

table required initially.

---

# Major Distributed Systems Insight

This pattern appears everywhere:

# Track POINTERS instead of INDIVIDUAL STATES

Examples:

| System        | Pointer                   |
| ------------- | ------------------------- |
| WhatsApp      | last_read_message_id      |
| Kafka         | consumer offset           |
| YouTube       | last watched timestamp    |
| Notifications | last_seen_notification_id |

This is a massive scalable systems pattern.

---

# The Real Goal of Schema Design

Schema design is NOT:

```text
thinking of tables
```

It is:

# modeling reality cleanly while minimizing future branching

---

# Final Mental Model

Whenever designing systems, repeatedly ask:

```text
1. What is the core thing?
2. What exists independently?
3. What connects things?
4. What happens over time?
5. Where does state belong?
6. Am I creating branching chaos?
```

That’s the foundation of senior backend/system design thinking.
