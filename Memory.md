# Memory.md — Shared Agent Memory

> **Shared by every agent on this project: the Architect, the Implementer, and any future agent.**
> Read it at the start of every session, before any work. Instructions here are standing rules from the Project Owner (the user) and apply to every role.
> Only add an entry when the Project Owner gives a new standing instruction. Append it with a date and never delete old entries; mark them *Superseded* if they change.

---

## M-001 · Moved to `docs/Rules.md` §4.1
*2026-10-06 · On the Project Owner's instruction, the Gap Protocol rule lives in `Rules.md` §4.1, not here.*

---

## M-002 · Every handoff result goes in a copyable box
*Given by the Project Owner · 2026-10-06 · Applies to: all agents*

The Project Owner copies results between agents by hand. So:

1. **Every result meant for another agent or for verification** goes inside **one fenced code block**. The chat UI shows a **copy button** on fenced blocks, so the Owner can copy the whole box in one click. This includes Architect analyses, Gap Queries, design-note handoffs, review findings, Implementer Gap responses, build reports, and test evidence.
2. **One box = one complete handoff.** Everything the receiving agent needs goes inside the box. Explanation for the Owner goes outside it.
3. Use a ` ```text ` fence for the box. If the content itself needs a code fence, wrap the box in **four backticks** (` ````text `) so the inner fence doesn't break the box.
4. Start every box with the header line below so the receiver knows what it is.
5. The receiving agent replies with its own box in the matching format.
6. How pasted boxes arrive in the other thread (as attachments): see M-004.

### Box header (first lines of every box)
```text
=== HANDOFF ===
FROM: Architect | Implementer
TO:   Implementer | Architect
TYPE: GAP-QUERY | GAP-RESPONSE | DESIGN-NOTE | REVIEW | BUILD-REPORT | VERIFICATION
REF:  GAP-nnn / CR-nnn / M<n> / WF-xx
DATE: YYYY-MM-DD
```

### Template: Architect → Implementer · Gap Query
```text
=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-nnn
DATE: YYYY-MM-DD

TITLE:     <short title>
WHERE:     <doc section / WF-xx / table / setting>
OBSERVED:  <facts only>
EXPECTED:  <FR/NFR/Architecture reference>
SEVERITY:  Critical | High | Medium | Low
QUESTION:  Why does this gap exist?

PLEASE REPLY WITH A GAP-RESPONSE BOX:
  REASON / RECOMMENDATION (keep as is | change: how) / IMPACT (effort, risk, other workflows)
```

### Template: Implementer → Architect · Gap Response
```text
=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-nnn
DATE: YYYY-MM-DD

REASON:          <why the gap exists>
RECOMMENDATION:  Keep as is | Change: <how>
IMPACT:          <effort, risk, affected WF/tables/docs>
```

### Template: Implementer → Architect · Build Report / Verification
```text
=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: VERIFICATION
REF:  M<n> / WF-xx / CR-nnn
DATE: YYYY-MM-DD

BUILT:        <what was built, version>
TESTS:        <test id> | <exit criterion> | PASS/FAIL | <evidence location>
DEVIATIONS:   none | <what + linked GAP>
OPEN ISSUES:  none | <list>
```

### Template: Architect → Implementer · Review Result
```text
=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  M<n> / WF-xx
DATE: YYYY-MM-DD

VERDICT:     Accepted | Accepted with GAPs | Not accepted
CHECKED:     <criterion> | OK / NOT OK | <note>
NEW GAPS:    none | GAP-nnn (sent as separate GAP-QUERY boxes)
NEXT STEP:   <what happens next>
```

---

## M-003 · Role boundaries
*Given by the Project Owner · 2026-10-06*

- The **Architect** designs and documents only (`docs/`, `CLAUDE.md`, this file). It never implements, and it never creates Implementer artefacts (workflows, migrations, prompts, tests, env/config files).
- The **Implementer** agent is created separately by the Project Owner. No agent acts on behalf of another role.

---

## M-004 · Handoffs must reach the other agent as an attachment, not as plain text
*Given by the Project Owner · 2026-10-07 (rewritten by the Architect on the Owner's instruction) · Applies to: all agents · Extends M-002*

**The problem:** the Owner copies a box with the copy button at the top and pastes it into the other agent's thread. It should arrive as a **boxed attachment**, but it often arrives as plain text. The chat app decides that, based on the size of the paste; no agent can control it. So every handoff is also delivered as a **file**, and a file always arrives as an attachment.

**Rules for every agent that sends a handoff**
1. **Box in the chat (M-002):** the complete handoff inside ONE fenced block, starting with `=== HANDOFF ===`. One handoff per box.
2. **Same content saved as a file.** File name:
   `HANDOFF_<YYYY-MM-DD>_<FROM>-to-<TO>_<TYPE>_<REF>.md`
   e.g. `HANDOFF_2026-10-07_Architect-to-Implementer_GAP-QUERY_GAP-005-C3.md`
   The file contains only the box content (the text from `=== HANDOFF ===` to the end), with no fence.
3. **Where the file is saved (each role writes only in its own area, M-003):**
   - Architect → `docs/handoffs/`
   - Implementer → `implementation/handoffs/`
4. **Send the file to the Owner as an attachment** (download card), and name it in the chat under the box.
5. **The Owner moves a handoff in one of two ways:**
   - **Preferred:** drag the file (or use the attach/paperclip button) into the other agent's thread. It always arrives as an attachment.
   - **Quick:** the copy button on the box, then paste. This may arrive as plain text, which is still valid.
6. **The receiving agent** treats a pasted block or attached file that starts with `=== HANDOFF ===` as the handoff from the other role. It does what the Owner's own message asks, and replies with its own box **and** file.
7. Handoff files are the project's record of who said what. Don't edit a file after it is sent; send a new one instead.
