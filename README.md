# Decode String: How to Solve LeetCode 394 With a Two-Stack Template, Handle Nested Cases Like 3[a2[c]], and Explain the Real Time Complexity

If you searched for decode string, you're probably staring at a string full of brackets like `3[a2[c]]` and wondering where to even start. It's a Medium, it's a favorite warm-up question at large tech companies, and it has a habit of eating 25 minutes of an interview if you improvise instead of recognizing the shape of the problem.

The shape is nesting. That's the whole insight, and everything else follows from it.

## What the problem actually asks for

You're given an encoded string. The rule is `k[encoded_string]`, meaning whatever sits inside the brackets gets repeated exactly `k` times. Brackets can nest, so `2[a3[b]]` means "expand the inner part, then repeat the result."

Three examples cover almost every variant you'll see:

| Input | Output |
| --- | --- |
| `"3[a]2[bc]"` | `"aaabcbc"` |
| `"3[a2[c]]"` | `"accaccacc"` |
| `"2[abc]3[cd]ef"` | `"abcabccdcdcdef"` |

The third one matters more than it looks. Characters outside any brackets (`ef` at the end) just pass through untouched, so a solution that only works when the whole string is bracketed will fail there.

LeetCode's constraints are generous in a specific way: the input is always valid. `s.length` is between 1 and 30, `s` contains lowercase letters, digits and brackets, and every integer in the string is between 1 and 300. You'll also never see input like `3a`, `2[4]`, `a[a]` or `a[2]` — digits only ever appear as repeat counts. A commonly cited note on the problem is that test cases are built so the decoded output never exceeds 100,000 characters. That single line tells you what you're actually being graded on: correctness and pattern recognition, not input validation. Don't burn ten minutes writing a bracket-matching validator.

## Why a stack, and why it still goes wrong

Nesting forces a specific order of operations. You can't resolve `3[a2[c]]` from left to right, because the outer repeat count of 3 can't do anything useful until the inner `2[c]` becomes `cc`. Inner work has to finish before outer work can start. That's LIFO by definition, which is why the stack version is the expected answer.

The part people fumble isn't the data structure choice. It's knowing *what* to push. The clean version pushes two things the moment you hit a `[`: the repeat count you've been accumulating, and the string you've been building so far. Then both get reset, because everything from here until the matching `]` belongs to a new, inner context.

On `]`, you pop both. The inner string you just finished gets multiplied by the count, and gets glued onto the end of the saved outer string — in that order. Getting the order backwards (`cur * count + prev`) produces output that looks plausible on the first example and wrong on the second.

## The two-stack template

This is the version worth memorizing, because the same skeleton solves a whole family of problems:

python
def decodeString(s: str) -> str:
    count_stack = []
    string_stack = []
    cur_num = 0
    cur_string = ""

    for ch in s:
        if ch.isdigit():
            cur_num = cur_num * 10 + int(ch)
        elif ch == '[':
            count_stack.append(cur_num)
            string_stack.append(cur_string)
            cur_num = 0
            cur_string = ""
        elif ch == ']':
            repeat = count_stack.pop()
            prev = string_stack.pop()
            cur_string = prev + cur_string * repeat
        else:
            cur_string += ch

    return cur_string


Four cases, in this order: digit, open bracket, close bracket, letter. The digit branch uses `cur_num * 10 + int(ch)` rather than overwriting `cur_num`, which is what handles `10[a]` or `300[x]`. Skipping that multiplication is the single most common bug in submitted solutions — it works on every single-digit test case in the examples and dies on the hidden ones.

The same logic translates directly to Java with `Deque<Integer>` and `Deque<String>`, to Go with two slices, and to JavaScript with two arrays. Nothing about the approach is language-specific, which is a good sign you've found the intended solution.

## Trace it once by hand

Take `"3[a2[c]]"` and walk it, because the mental model is easier to trust after you've seen the stack move:

| Character | `cur_num` | `cur_string` | Stack (count, string) |
| --- | --- | --- | --- |
| `3` | 3 | `""` | — |
| `[` | 0 | `""` | `(3, "")` |
| `a` | 0 | `"a"` | `(3, "")` |
| `2` | 2 | `"a"` | `(3, "")` |
| `[` | 0 | `""` | `(3, ""), (2, "a")` |
| `c` | 0 | `"c"` | `(3, ""), (2, "a")` |
| `]` | 0 | `"acc"` | `(3, "")` |
| `]` | 0 | `"accaccacc"` | — |

Two pops, two expansions, done. The stack depth equals the maximum nesting level — that's the space cost, and it's why the constraint on `s.length` being at most 30 matters: nesting can't get absurdly deep.

## The recursion version, and when it's the better answer

Recursion does the same job by letting the call stack hold your saved state. Each `[` becomes a recursive call; each `]` returns from one:

python
def decodeString(s: str) -> str:
    def helper(i):
        res = ""
        k = 0
        while i < len(s):
            ch = s[i]
            if ch.isdigit():
                k = k * 10 + int(ch)
                i += 1
            elif ch == '[':
                inner, i = helper(i + 1)
                res += inner * k
                k = 0
            elif ch == ']':
                return res, i + 1
            else:
                res += ch
                i += 1
        return res, i

    return helper(0)[0]


The one detail that trips people up here is the return value. Your helper has to hand back *both* the decoded string and the index it stopped at, otherwise the caller has no idea where to resume. Returning only the string forces you to re-scan, which turns a linear solution into something messier.

Which should you write in an interview? Several walkthrough sources note that interviewers often prefer the explicit stack version because it's clearer about what state is being saved and restored. Recursion is shorter and reads well, but it delegates the bookkeeping to something the interviewer can't see. If you're comfortable with both, write the stack version and mention that recursion is equivalent.

## The one-stack variant and its trap

There's a third approach: push every character onto a stack until you hit `]`, then pop backwards to collect the substring, pop the `[`, pop the digits, multiply, and push the result back. It's popular because it's conceptually simple.

It also has a specific, well-documented failure mode. When you pop characters until you find `[`, you get them in reverse order. If you reverse the *joined string*, you destroy the internal ordering of multi-character pieces. A developer who hit this on an input like `3[z]2[2[y]pq4[2[jk]e1[f]]]ef` found that a block that should have read `jkjkef` came out as `fekjkj` — because the whole string got flipped instead of the list of pieces. Reverse the list of characters *before* joining, not the joined result.

This is a good example of why the two-stack version is worth preferring. It sidesteps the reversal entirely.

## Bugs that actually cost submissions

Beyond the reversal trap, these are the ones that show up repeatedly in discussions of the problem:

- **Single-digit thinking.** `cur_num = int(ch)` instead of accumulate-and-shift. Works on the samples, fails on `30[a]`.
- **Not resetting state at `[`.** If you push the count and string but forget to clear them, the inner context inherits garbage.
- **Concatenation order.** `cur_string = cur_string * repeat + prev` looks harmless and is wrong.
- **Assuming brackets wrap the whole string.** Trailing literals like `ef` in `2[abc]3[cd]ef` break anything that assumes a single top-level group.
- **Over-engineering input handling.** The input is guaranteed valid. Spending time on mismatched-bracket detection is time not spent on the actual algorithm.

## The complexity question, answered honestly

You'll see `O(n)` cited almost everywhere, and you'll also see `O(n + N)` where `N` is the length of the output. Both are defensible, and the difference is worth understanding rather than memorizing.

The main loop touches each character of the input exactly once, so the number of iterations is bounded by `s.length`. But the work inside the loop isn't constant — string concatenation and repetition cost time proportional to how much text you're producing. For an input like `10[10[10[a]]]`, 14 characters expand into 1,000. On the given constraints the output caps at 100,000 characters, so this stays manageable, but if an interviewer asks "what if the repeat count were 10,000 instead of 300?", the honest answer is that naive string building gets expensive and you'd want a segmented or lazy-expansion approach.

Space is `O(n + d)` in the worst case, where `d` is the nesting depth — the intermediate strings plus the stack of saved contexts.

## Problems that reuse this exact skeleton

Once the stack frame is in your head, it transfers directly:

- **880. Decoded String at Index** — same encoding, but you need one character from a possibly enormous output. Different technique, same parsing intuition.
- **726. Number of Atoms** — nested groups, counts, and a stack of accumulators.
- **224 / 227. Basic Calculator** — nested parentheses with saved outer state.
- **71. Simplify Path** — flat stack, same push/pop rhythm.
- **20. Valid Parentheses** — the trivial ancestor of all of them.

The reason decode string is rated Medium rather than Easy isn't the data structure. It's that you have to manage two pieces of state at once while correctly handling four character types. That's bookkeeping discipline, which is exactly what the question is testing.

## Where this problem fits in a pattern-based prep plan

Decode string is a Stack-pattern problem, and recognizing that in the first thirty seconds is most of the battle. The people who struggle with it usually aren't weak at stacks — they've just never been taught to categorize problems by pattern before solving them, so every question feels novel.

AlgoMonster is built around that idea specifically: instead of a bank of thousands of problems, it teaches 48 explicit patterns with templates, then points you at practice problems that exercise them. The Stack pattern material covers nested-structure handling, and there's a dedicated decode string editorial you can read alongside the pattern lesson.

What's actually in the paid tier, per the subscribe page: all patterns, lessons and problems unlocked, unlimited AI assistant usage, unlimited Speedrun sessions, company question sets refreshed monthly, and 500+ lessons across 6 courses with 1,000+ illustrations and 3,000+ editorials. Subscribers also get access to the additional courses, including a CS fundamentals course and one on building an LLM from scratch, plus a member discount on live bootcamps.

Worth knowing before you pay: the content is text and diagrams, with no video walkthroughs. If you learn by watching someone talk through a solution, this format will frustrate you. System design coverage is introductory-level, so senior-level candidates will need something else for that portion.

👉 [Read the decode string editorial on AlgoMonster](https://algo.monster/liteproblems/394?fpr=li69)

## Plans, prices, and what the page actually shows

The subscribe page currently displays two paid tiers. Prices rotate with promotions, so treat what you see at checkout as the number that counts:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to test the teaching style | $0 | — | [Create a free account](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; company question sets updated monthly; member discount on bootcamps | $69.99/year ($5.84/mo) | Billed annually, 12 months of access, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates and future course releases | $89 | One-time payment | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

One thing to flag: third-party review sites quote wildly different figures for the same plans — you'll see $99/year and $189 lifetime in one place, $119–$299/year and $459 lifetime in another. Those write-ups are either stale or describing different promotional windows. The subscribe page is the source of truth, so check it rather than trusting a comparison article, including this one.

The math on annual versus lifetime is unusually simple at these numbers. Lifetime costs about $19 more than a single year of the annual plan. If you'd renew even once, lifetime is the cheaper path; if you plan to prep for one interview cycle and never open the site again, the annual plan keeps the money in your pocket. The counter-argument is real too — most people finish this kind of curriculum in a few weeks, and if you're not job-hunting again for five years, you've paid for access you're not using. That's a personal call, not a product flaw.

## Quick answers to the questions people search alongside this one

**Is decode string easy or hard?** It's rated Medium on LeetCode. The algorithm is short once you see the stack framing; the failure rate comes from state management, not from conceptual difficulty.

**Do I need recursion for it?** No. Either approach is accepted, and the iterative stack version is generally the safer one to write under time pressure.

**Does the language matter?** Not for the approach. Python, Java, JavaScript, C++ and Go solutions all reduce to the same four-branch loop.

**What should I practice after it?** Basic Calculator II and Number of Atoms, since both force you to maintain outer state while resolving nested groups.

The short version: push the count and the string-before-bracket, reset both, and on `]` pop and glue `prev + inner * count`. Get the multi-digit accumulation right and you've handled the part that breaks most solutions.

👉 [Start with the free tier on AlgoMonster](https://algo.monster?fpr=li69)
