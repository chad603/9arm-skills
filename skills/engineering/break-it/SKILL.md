---
name: break-it
description: Adversarial test-and-assertion discipline — your test's job is to falsify "this code is correct," not to confirm it works. Test the contract not the implementation, attack the edges not the happy path, watch every test fail before you trust it, lock every fixed bug with a regression test at the seam where it hid, and assert invariants in-code so corruption fails loud at the source. Trigger on /break-it and proactively whenever correctness needs locking down — user asks to write/add tests, asks "how do I test this", is about to trust a change with no failing test behind it, just fixed a bug that needs a regression test, or asks whether code is actually correct.
---

# Break It

Write the test that tries to **break** the code, not the one that confirms it works. If you can't break it, *then* you trust it. Same adversarial stance as [`debug-mantra`](./debug-mantra/SKILL.md) ("what would disprove it?") and [`scrutinize`](./scrutinize/SKILL.md) ("read it cold") — pointed at the test suite. A passing test proves nothing until you've seen it fail for the right reason.

## Operating stance

- **A test is a falsification attempt.** Its purpose is to find the input that makes the code wrong. A test that can only pass is decoration.
- **Never-red means never-trusted.** A test you have not watched go red has never been shown to assert anything. Make it fail first, on purpose.
- **The bug is at the edge, not the middle.** Happy-path inputs are where bugs aren't. Spend your attention on boundaries.

## The discipline — apply in order

> **Break-it mantra:**
> 1. **State the contract.** What must always hold? Test that, not how it's coded.
> 2. **Attack the edges.** Empty, one, full, max, overflow, null, dup, out-of-order, concurrent.
> 3. **See it fail first.** Red for the right reason, then green. Never-red proves nothing.
> 4. **Lock the bug.** One bug → one regression test, placed at the seam where it hid.
> 5. **Assert the invariant.** Encode "this must hold" in-code so violation fails loud at the source.

---

### 1. State the contract

Before writing a test, write down what the code *promises* — in one sentence.

- **Preconditions, postconditions, invariants.** "After `push` returns true, the item is retrievable in FIFO order, and no unread item is ever overwritten." That sentence *is* your test plan.
- **Test the contract, not the implementation.** Assert on observable behavior (return values, side effects, state transitions), not on internal variables or call counts. A test bound to the implementation rots on the next refactor *and* still misses the bug — the worst of both.
- If you cannot state the contract, you don't understand the code well enough to test it. Trace it first (`scrutinize` step 2) or the tests will assert the wrong thing.

### 2. Attack the edges

The middle of the input space is where code is obviously right. Bugs live at the boundaries. Walk this checklist for every input:

- **Size boundaries:** empty / zero, exactly one, exactly capacity, capacity + 1, max, overflow past max.
- **Value boundaries:** null / None, negative, zero, the sentinel value, the max representable.
- **Shape boundaries:** duplicates, already-sorted, reverse-sorted, out-of-order arrival, unicode / multibyte, embedded null, huge.
- **State boundaries:** first call, re-entry, after error, after close, partial failure then retry.
- **Order / concurrency:** two callers racing, interleaved push/pop, reordering, the operation interrupted mid-flight.

You don't need all of these every time — you need the two or three that this contract is most likely to violate. Pick them deliberately, not by reflex.

### 3. See it fail first

A green test you've never seen red is a liability: it might assert nothing, test the wrong path, or be silently skipped.

- **Make it red on purpose.** Either write the test before the fix (and watch it fail against the broken code), or temporarily break the code and confirm the test catches it. Red for the **right reason** — read the failure message, make sure it's failing on the assertion you care about, not on a setup error.
- Then make it green. Red → green is the proof the test has teeth. Green-only is theater.
- **Don't mock away the thing under test.** A mock that stands in for the buggy code guarantees the bug walks straight through the test. Mock the *boundary* (network, clock, disk), never the unit you're trying to break. (`scrutinize` flags this too — tests that pass while skipping the traced path.)

### 4. Lock the bug

Every bug found — by `debug-mantra`, by review, by production — earns a regression test before the fix is called done.

- **Write the test that *would have caught it*.** Not a test near the bug — the exact input that triggers it, asserting the now-correct behavior.
- **Place it at the seam where it hid.** If the bug was a latent path nothing exercised, the test goes at that path. The regression test's job is to make this exact class fail loudly forever after.
- This is the test that feeds `post-mortem`'s action items and `debug-mantra`'s "failing test as repro." One bug, one locked door.

### 5. Assert the invariant

Tests check correctness at test time. Assertions check it at *every* run.

- For a property that must **always** hold (`count <= capacity`, `head != null after init`, `balance never negative`), encode it as an in-code assertion / precondition check at the source.
- **Fail loud, not silent.** A violated invariant should crash at the line where the corruption happens — not silently propagate and surface three layers downstream as a mystery. (This is exactly the `post-mortem` lesson: a null-check added "so the same class of bug fails loudly instead of silently spinning.")
- **Make illegal states unrepresentable** where the language allows it — a type, an enum, a constructor that rejects bad input beats an assertion that catches it later. The cheapest bug is the one the compiler refuses to compile.

---

## When to invoke

- `/break-it`
- "write tests / add test coverage / how do I test this"
- "is this actually correct? / how do I know this is right?"
- Right after `debug-mantra` lands a fix — the regression test is part of done.
- Proactively when someone is about to trust a non-trivial change with no failing test behind it.

## When NOT to use

- **Reviewing existing tests for whether they're any good** → that's [`scrutinize`](./scrutinize/SKILL.md) ("how is it tested — do the tests exercise the path or skip it?"). This skill *writes* the tests; scrutinize *judges* them.
- **Chasing a live bug** → [`debug-mantra`](./debug-mantra/SKILL.md) first. Reproduce and locate it, then come here to lock it.
- **Testing what the type system already guarantees.** Don't write a test asserting a non-null field is non-null when the type forbids null. Spend the effort on what the compiler can't prove.

## Operating rules

- **Never trust a test you haven't seen fail.** Never-red is never-verified. Step 3 is not optional.
- **Test the contract, not the implementation.** Assert observable behavior. A test coupled to internals rots on refactor and still misses bugs.
- **The happy path is the least valuable test.** If you only have time for a few tests, spend them on edges. The middle rarely breaks.
- **One bug, one regression test, at the seam.** A fix without the test that locks it is not done.
- **Don't mock the unit under test.** Mock the boundary. A mock over the buggy code lets the bug pass green.
- **Coverage % is not correctness.** 100% line coverage with weak assertions catches nothing. Assertions are the test; the lines are just how you reach them.
- **Assert invariants in-code and fail loud.** A correctness violation should crash at its source, not corrupt quietly and surface elsewhere.
- **Prefer making illegal states unrepresentable** over testing that they don't occur. The compiler is the cheapest test in the suite.

## Worked example — ring buffer that loses data at wraparound

> **1. Contract.** `RingBuffer(cap)`: after `push(x)` returns `true`, `x` is retrievable by `pop()` in FIFO order; `push` on a full buffer returns `false` and overwrites nothing; `pop` on empty returns `none`. Invariant: `0 <= count <= cap`, and full vs. empty are always distinguishable.
>
> **2. Attack the edges.** The dangerous boundary is *exactly full, then wrap*. The implementation tracks only `head` and `tail` indices and calls the buffer empty when `head == tail` — but `head == tail` is *also* what a completely full buffer looks like after wraparound. That ambiguity is the bug. Happy-path "push 3, pop 3" never reaches it.
>
> **3. See it fail first.** Test: fill to capacity (`cap` pushes, all expected `true`), assert the `cap+1`-th `push` returns `false`, then `pop` exactly `cap` times and assert the values come back in order. Run against the buggy code → **red**: after filling to capacity, `head == tail`, so `pop` reports empty and returns `none` — the data is silently lost. Failure message confirms it's the FIFO assertion failing, not a setup typo. Fix (add an explicit `count`, or a `full` flag) → **green**.
>
> **4. Lock the bug.** Regression test `test_push_pop_at_capacity_wraps` placed exactly at the fill-to-capacity-then-wrap seam — the path no happy-path test exercised. This is the test that would have caught it, and now fails loudly forever if anyone reintroduces the `head == tail` ambiguity.
>
> **5. Assert the invariant.** Added `assert 0 <= count <= cap` at the top of both `push` and `pop`, so any future index-arithmetic bug that corrupts `count` crashes at the source instead of silently losing data downstream. Better still: `count` is now the single source of truth for full/empty, so the ambiguous state is no longer representable.

What this did that a happy-path suite wouldn't: it found the *exactly-full wraparound* boundary by reasoning about the contract, watched the test go red against the real bug (proving it has teeth), locked the seam, and turned the silent data-loss into a loud assertion failure.

## Handoffs

- **Came from a debug session?** [`debug-mantra`](./debug-mantra/SKILL.md) found the bug and built the repro — turn that repro into the locked regression test here (step 4).
- **Writing up the fix?** [`post-mortem`](./post-mortem/SKILL.md) wants the regression test name and seam in its action items — hand them over.
- **Want a second opinion on the suite you just wrote?** [`scrutinize`](./scrutinize/SKILL.md) reads it cold and asks whether the tests exercise the real path or pass while skipping it.
