# Is Subsequence: The O(n) Two-Pointer Check, the Edge Cases That Fail Your Submission, and the Follow-Up Interviewers Actually Ask

Most people who land on "is subsequence" can already explain the answer out loud. Walk through `t`, and every time a character matches the next character you still need from `s`, tick one off. The bugs show up somewhere else: in the loop condition, in the empty-string case, or in the follow-up where `t` never changes and a billion candidate strings come at you one at a time.

So this is 392 from the ground up. What the problem asks, the two-pointer check with a real walkthrough, the edge cases behind most wrong answers, and the preprocessing trick for the follow-up. Then the part editorials usually skip: why this "Easy" is really a pattern-recognition test, and what that means for how you should be practicing.

## What "is subsequence" is actually asking

You get two strings, `s` and `t`. Return `true` if `s` is a subsequence of `t`, otherwise `false`.

A subsequence is what's left after you delete zero or more characters from the original string without changing the relative order of the ones that remain. Delete `b` and `d` from `"abcde"` and you get `"ace"`, so `"ace"` is a subsequence of `"abcde"`.

The LeetCode version currently caps `s` at length 100 and `t` at length 10,000, both lowercase English letters. An older version of the statement used a `t` of roughly 500,000 characters so the second pointer mattered more.

Two examples cover the whole logic:

| `s` | `t` | Output | Why |
| --- | --- | --- | --- |
| `"abc"` | `"ahbgdc"` | `true` | Matches land at positions 0, 2, 5 |
| `"axc"` | `"ahbgdc"` | `false` | `x` never appears in `t` |

### Subsequence is not substring, and it is not "same set of letters"

This is where the problem stops being about strings and starts being about reading carefully.

`"ace"` is a subsequence of `"abcde"` but not a substring, because subsequences are allowed to skip. The other direction is cleaner: every substring is a subsequence, but not the reverse. A single `t.indexOf(s)` call only answers the substring question, so it's useless here.

The second trap is set-based thinking. `"aec"` contains the same letters as `"ace"`, and every one of those letters appears in `"abcde"`, yet the answer is `false`, because `e` shows up at index 4 and `c` at index 2. Order is the constraint, not membership. If your solution counts character frequencies, it will pass a few samples and fail on exactly this shape of input.

## The two-pointer check

Two indices, one per string. `j` sweeps `t` from left to right and always moves forward. `i` only moves when `s[i]` and `t[j]` agree. If `i` reaches the end of `s`, you matched everything in order.

python
class Solution:
    def isSubsequence(self, s: str, t: str) -> bool:
        i = j = 0
        while i < len(s) and j < len(t):
            if s[i] == t[j]:
                i += 1
            j += 1
        return i == len(s)


That's the whole thing. No recursion, no table, no memoization. Time is `O(|s| + |t|)` because you touch each string once, and extra space is `O(1)` for the two integers. If `len(t) < len(s)` you can return `false` immediately, though the loop already handles it.

Here is the pointer state for `s = "ace"`, `t = "abcde"`:

| Step | `t[j]` | Looking for | Match | After |
| --- | --- | --- | --- | --- |
| 1 | `a` | `a` | yes | `i=1`, `j=1` |
| 2 | `b` | `c` | no | `i=1`, `j=2` |
| 3 | `c` | `c` | yes | `i=2`, `j=3` |
| 4 | `d` | `e` | no | `i=2`, `j=4` |
| 5 | `e` | `e` | yes | `i=3`, loop ends |

`i == 3 == len(s)`, so the answer is `true`. The matched positions in `t` are 0, 2 and 4. Losing track of "advance `j` on a mismatch but not `i`" is the single most common implementation slip in this problem.

### Why the greedy choice is safe

There's a legitimate worry buried here: what if matching an early occurrence of a character costs you a later one you needed?

It can't, and the argument is short. Say the current character you need appears at indices 3 and 7 in the remaining part of `t`. Taking index 3 leaves you everything from index 4 onward to work with. Taking index 7 leaves you only what's after 7, which is a strict subset of that. Whichever solution used 7 can be rewritten to use 3 and stay valid, because everything after 7 is still after 3. Taking the earliest match never removes an option.

That's the same reasoning that makes greedy safe in a lot of string matching, and it's worth being able to say out loud in an interview rather than just asserting it.

## The five things that break correct-looking code

**Empty `s`.** If `s` is `""`, the loop never executes and `i` stays at 0, which equals `len(s)`. The function correctly returns `true`, with no special case needed. LeetCode does include this input. An empty string is a subsequence of everything, including an empty `t`.

**Empty `t` with a non-empty `s`.** Same loop, same logic: `i` stays below `len(s)` and you return `false`. Also correct without a special case.

**Using `or` instead of `and` in the loop condition.** This one throws an `IndexError` rather than a wrong answer, which is at least honest of it. With `or`, the loop keeps running after one pointer hits the end and you index past a boundary.

**Advancing `j` only inside the match branch.** The mismatch case then makes no progress and you spin or return the wrong result. `j` advances on every iteration, unconditionally.

**The `for` loop variant without a bounds guard.** Writing `for c in t: if s[i] == c: i += 1` crashes once `i` reaches `len(s)`, because `s[i]` is out of range. Either keep the `while` form or add `i < len(s) and` to the condition.

One non-bug worth ignoring: adding an early `return True` the moment `i == len(s)` inside the loop. It changes nothing about complexity and makes the function harder to read. The single return at the bottom is fine.

## The follow-up: one fixed `t`, a billion incoming strings

The follow-up asks what changes if you have `s₁, s₂, … sₖ` with `k >= 10^9` and you check each one against the same string `t`.

Running the two-pointer scan per query is now the wrong shape of solution. With `t` at 10,000 characters and a billion queries, you're looking at roughly 10^13 character comparisons. Even at a nanosecond each, that's measured in hours, and the actual cost is worse.

The fix is to stop treating `t` as something you scan and start treating it as something you preprocess. Because `t` never changes, everything you learn about it can be reused.

**Approach one: position lists plus binary search.** Record, for each of the 26 letters, the sorted list of indices where it appears in `t`. To answer a query, keep a `current` position and, for each character of `s`, binary search that character's list for the smallest index greater than `current`. Preprocessing is `O(|t|)`; each query becomes `O(|s| log |t|)`. The logarithm is over 10,000, so about 14 steps per character of `s`, and `s` is capped at 100. This is the answer most interviewers are fishing for.

**Approach two: a next-occurrence table.** Build `store[j][c]` holding the smallest index `>= j` where character `c` appears in `t`, filling it right to left. Preprocessing is `O(26 × |t|)` in time and space, and each query drops to `O(|s|)` with no binary search at all. More memory, constant-time jumps.

| Approach | Preprocessing | Per query | Extra space |
| --- | --- | --- | --- |
| Two pointers per query | none | `O(\|t\|)` | `O(1)` |
| Position lists + binary search | `O(\|t\|)` | `O(\|s\| log \|t\|)` | `O(\|t\|)` |
| Next-occurrence table | `O(26 × \|t\|)` | `O(\|s\|)` | `O(26 × \|t\|)` |

Practically, the binary-search version is the one you can write under pressure without a diagram. The table version is what you mention as an optimization once the first one is working. The general name for this idea is a subsequence automaton, and the identical structure shows up as an actual problem in 792. Number of Matching Subsequences, where you get thousands of words instead of billions.

## Subsequence, substring, and the DP relatives

LeetCode tags 392 with Dynamic Programming alongside Two Pointers, Greedy and Divide & Conquer. People reasonably ask why, given that the two-pointer solution never builds a table.

The DP framing is real, and it's this: define `dp[i][j]` as whether `s[i:]` is a subsequence of `t[j:]`. If the characters match, `dp[i][j] = dp[i+1][j+1]`; otherwise `dp[i][j] = dp[i][j+1]`. Base case: any suffix of `s` starting at `len(s)` is trivially a subsequence of anything. It works, it runs in `O(|s| · |t|)` time and space, and it's strictly worse than two pointers for a yes/no answer.

It's still worth knowing because it's the doorway to the rest of the family:

- **1143. Longest Common Subsequence** is where 2D DP becomes mandatory, not optional.
- **583. Delete Operation for Two Strings** is LCS with a twist on what you count.
- **115. Distinct Subsequences** asks how many ways, which forces a count instead of a boolean.
- **792. Number of Matching Subsequences** is the follow-up dressed up as its own problem.

If you're going to spend time on this area, the return on the DP branch is much higher in 1143 than in 392. Anyone who tells you to solve 392 with a DP table is technically correct and practically wasting your afternoon.

## What this problem is really testing

392 is classified Easy, and the code takes eight lines. The reason it still shows up in interviews is that it's a cheap filter for three habits: whether you reach for the right tool before writing anything, whether you check your loop bounds and empty inputs voluntarily, and whether you can extend a working solution when the constraints move instead of freezing.

That third habit is the one that separates candidates. The follow-up isn't a trick question with a gotcha answer. It's a straightforward "the bottleneck moved, what do you do about it," and the correct response is to name the bottleneck out loud, then preprocess.

Which is also the honest pitch for pattern-based learning. The difference between someone who solves 392 in four minutes and someone who stares at it for twenty usually isn't string knowledge. It's whether "order-preserving match across two sequences with skips" fires the two-pointer association on sight.

AlgoMonster builds its curriculum around exactly that layer. It's a text-and-diagram course rather than a video library, written by engineers who came out of Google, and it organizes material around roughly 48 recurring patterns with decision flowcharts for picking between them. Their walkthrough of this specific problem is free to read and covers the intuition, the pointer trace, implementations in several languages, and a pitfalls section: 👉 [Read the full Is Subsequence walkthrough and pitfalls](https://algo.monster/liteproblems/392?fpr=li69).

The flowchart is the more transferable piece. It asks things like "small constraints?", "sorted input or monotonic answer?", "need O(1) memory?" and branches toward two pointers, sliding window, binary search, or a hash map depending on the answers. Applied to this problem, "two sequences, order matters, constant extra memory" lands on two pointers without much debate: 👉 [See the pattern decision flowchart](https://algo.monster/flowchart?fpr=li69).

Worth being clear about what it isn't. Third-party reviews of the platform flag the same trade-offs repeatedly: no video explanations at all, a problem bank in the low hundreds rather than LeetCode's 3,000+, introductory-level system design coverage, and light behavioral content. If you need videos or a huge problem volume, this isn't the tool, and reviewers say so themselves.

## AlgoMonster plans and what they cost

The platform has three tiers and no monthly option, which is unusual in this category and worth knowing before you go looking for one.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns, lessons and problems. Enough to judge whether the text-and-diagram format suits you | $0 | No card, no term | [Start on the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems unlocked. Unlimited AI Assistant usage. Unlimited Speedrun. Company questions updated monthly. 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials. Member discount on live bootcamps. 12 months of access, cancel or upgrade anytime | $99/year, shown as $8.25/mo billed annually | Annual | [Get Pro Yearly access](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates. According to the checkout page, an internal referral to a top tech company for qualified candidates | $189 | One-time payment, yours permanently | [Get Pro Lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two things to check yourself before paying, because they change:

The yearly and lifetime prices on the subscribe page move with promotions. Across snapshots of the official page, yearly has appeared at both $99/year and, during a deeper sale, $69.99/year, with lifetime at $189 and at one point $89. Third-party blogs quote all kinds of other numbers, including $119 to $299 per year and a $459 lifetime price, which don't match what the official checkout page currently shows. Treat anything you read on a coupon site as stale until the checkout page confirms it.

There's no formal free trial beyond the free tier. If you want to know whether the teaching style works, the free content is the preview, and a handful of editorial pages like the 392 walkthrough cost nothing to read.

### Which tier makes sense

If you came here for this one problem and you're done after it, the free editorial is enough and you should close the tab.

If you're in a compressed prep window, the annual plan is the lower-risk commitment. At $99 it undercuts LeetCode Premium, and the absence of a monthly option means there's no trickle of $35 charges while you decide.

Lifetime is a bet on repetition. At $189 against $99 annual, it pays for itself just under two years in, which matters if you interview repeatedly rather than once. Reviewers who recommend it frame the argument the same way: engineers who change jobs every two or three years and refresh their prep each time are the audience for a one-time payment.

What no plan fixes is effort. Anyone who buys lifetime access and doesn't work through the patterns has spent $189 on a bookmark.

## Quick answers

**Is an empty string a subsequence of anything?** Yes. `""` is a subsequence of every string, including `""`.

**Is every substring a subsequence?** Yes, since contiguous characters are already in order. The reverse isn't true: `"ace"` is a subsequence of `"abcde"` but not a substring.

**Can I check this with a hash map?** Only for the "does `t` contain all these characters" question, which isn't what's being asked. Order-dependence means `"aec"` fails against `"abcde"` regardless of what the map says.

**What's the complexity of the two-pointer solution?** `O(|s| + |t|)` time and `O(1)` extra space. No solution does better asymptotically on a single query, because in the worst case you have to look at the whole of `t`.

**Why is 392 tagged Dynamic Programming?** Because the problem has a valid 2D DP formulation, and it connects to the LCS family. Two pointers is still the better answer for the actual question being asked.

**Does AlgoMonster have a monthly plan?** No. Free, yearly, or lifetime.

If the pattern angle is what you're missing rather than the strings, the fastest path from here is to read the free editorial, then run a handful of two-pointer problems back to back while the shape is fresh. The curated lists on the platform group them for exactly that purpose: 👉 [Browse the curated practice list](https://algo.monster/practice/monster50?fpr=li69). And if you decide to go past the free tier, 👉 [compare the yearly and lifetime plans directly on the checkout page](https://algo.monster/subscribe?fpr=li69), since promotions there decide the real number.
