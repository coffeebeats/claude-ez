# ez:plan — on the pick, write the plan and build it

Read by `/ez:plan` when a direction is picked: by the user's reply to the
options, or in an unattended run by its build test. `SKILL.md` holds the
hard rules, which still apply, and names `<plan-file>` and `<git-dir>`.

## The reply

The user's reply decides what happens:

- **A number, with or without amendments.** Fold amendments of detail into
  the plan. An amendment that changes the mechanism, what the direction
  builds on, or a decision it marks one-way makes a direction nobody judged:
  say so, dispatch the proposer told the amendment is decided, put the
  evaluator on what comes back as in Phase 4, print that direction in the
  options' shape, and wait.
- **`hold` and a number.** Write the plan and stop, naming the next step:
  reply `go` here to build it below, or `/clear` and ask to build the plan in
  `<plan-file>`, printed as its full path, which carries its own build rules.
- **`go`, after a hold.** Build the held plan as below.
- **`none` and what to change.** Run Phase 4 again with the user's words as
  preferences, or, where they are constraints, added to the frame in the plan
  file, and present again. A round the user asks for is theirs to ask for;
  the two-round limit is on rounds that run on their own.
- **`more` and a number.** Print that direction's cut detail from the
  agents' returns, as labeled sub-bullets (How, Builds on, Touches, Risks,
  Verify, two-way decisions, and the evaluator's verdict on each check), and
  wait.
- **A question.** Answer it from what the agents returned, and wait.

## The plan section

To write the plan, Read `<plan-file>`, change `framed` in its
heading to `picked`, and add this section after the brief, replacing any
earlier one:

```markdown
## Plan — <name>
Picked from <n> directions on <date>; evaluator: <verdict>.

Context: <the problem and why it matters, with the issue reference>. Done when: <the criteria>.
Approach: <the mechanism and how it works, with the user's amendments folded in>.
Builds on: <path:line — what it contributes>; New: <piece — why nothing existing will do>.
Touches: <files or modules>.
Decisions: <decision (one-way | two-way)>; …
Conditions: <each gap the evaluator named, as something that must hold>.
Not doing: <the non-goals>; <each direction not taken — why, in a clause>.
Verify: <tests to extend or add; the end-to-end check>.
Left to implementation: <named>.
Build rules: read each file before changing it; stop and tell the user, without re-planning, when the code contradicts **Builds on**, a caller the plan missed, or a constraint it missed; meet every **Condition**, running the narrowest test as each part lands and the **Verify** check at the end; close by checking each **Done when** and **Condition**, changing `picked` in this file's heading to `built` only when every one is met, then name `/ez:review`.
```

## The build

Then build it, in this session, from that section rather than from the
options printed earlier:

1. Read each file before changing it. This context has not read the code, and
   what it reads now is the tree as it stands.
2. When the code contradicts the plan — a component does not do what
   **Builds on** says, a caller the plan did not account for, a constraint
   it missed — stop and tell the user what the plan expected, what you
   found, why it matters, and the options. Do not re-plan silently. A detail
   the plan left to implementation is not a contradiction.
3. Meet every **Condition**. As each part lands, run the narrowest existing
   test that exercises it, and run the end-to-end check from **Verify** at
   the end. Say what ran and what it printed, or that nothing could run and
   why.
4. If this context is compacted or cleared partway through, read the plan
   section again before going on.

Close by holding the work against the plan, in this shape:

```markdown
**Built: <name>**
- `met` <Done when criterion> — <the check that shows it>
- `NOT MET` <Condition> — <what is missing>
- `departed` <what changed from the plan> — <why>
```

One line per **Done when** criterion and per **Condition**, then one per
departure from the plan. When every one is met, Read `<plan-file>` and change
`picked` in its heading to `built`; otherwise leave it `picked`, so the next
run asks before replacing it. Then name the next step in one line:
`/ez:review`, which reads the change critically before it ships.
