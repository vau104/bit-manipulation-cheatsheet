# Bit Manipulation Cheat Sheet: Operators, Tricks, and Bitmask Formulas That Cover Most Interview Problems

Bit manipulation has a reputation problem. It gets taught as a bag of disconnected tricks — forty slides of XOR, someone mutters "n & (n-1) matters," and nothing sticks because there's no spine holding it together.

There is a spine. Nearly every bit problem in a coding interview lands in one of five buckets: single-bit masking, the two low-bit idioms, XOR folding, popcount, and bitmask enumeration. That's it. Get those five in your hands and you stop memorizing and start recognizing.

What follows is the cheat sheet I'd actually want pinned above a desk: the operators, the formulas with binary traces so you can see *why* they work, the problem numbers they map to, and a section on the part most guides skip — knowing which trick applies before you've written a line of code.

## The six operators, and what each one is for

You only need two of these for 90% of interview problems (AND and XOR), but the other four show up the moment you write a mask.

| Operator | Name | Rule | Example (4-bit) | What it's for |
| --- | --- | --- | --- | --- |
| `&` | AND | 1 only if both bits are 1 | `1010 & 0110` = `0010` | Testing bits, masking |
| `\|` | OR | 1 if either bit is 1 | `1010 \| 0110` = `1110` | Setting bits, merging flags |
| `^` | XOR | 1 if bits differ | `1010 ^ 0110` = `1100` | Toggling, cancelling pairs |
| `~` | NOT | Flips every bit | `~1010` = `...0101` | Building masks |
| `<<` | Left shift | Moves bits left, pads zeros | `5 << 1` = `10` | Multiply by 2^k |
| `>>` | Right shift | Moves bits right | `5 >> 1` = `2` | Divide by 2^k (floor) |

Two things worth saying out loud about shifts. `x << k` is multiplication by 2^k and `x >> k` is floor division by 2^k — both true only while you don't overflow. In Python, integers are unbounded, so there's no overflow and no natural 32-bit wraparound; if a problem says "simulate a 32-bit register," you have to mask manually with `& 0xFFFFFFFF`. That's a real interview trap, not a footnote.

And `~a` equals `-a - 1` in every language using two's complement. That identity is what makes the next trick work, so it's worth ten seconds of thought rather than a memorized line.

## Setting, clearing, toggling, testing: the four masks

Every single-bit operation follows the same recipe: build a mask with one bit in position `i` (`1 << i`), then combine it with the right operator.

| Goal | Code | Why it works |
| --- | --- | --- |
| Test bit `i` | `(x >> i) & 1` | Shift the bit to position 0, mask off everything else |
| Set bit `i` to 1 | `x \| (1 << i)` | OR forces that bit to 1, leaves the rest untouched |
| Clear bit `i` to 0 | `x & ~(1 << i)` | AND with the inverted mask zeroes exactly one bit |
| Toggle bit `i` | `x ^ (1 << i)` | XOR flips it, everything else is unchanged |

In Python:

python
def test_bit(x: int, i: int) -> int:  return (x >> i) & 1
def set_bit(x: int, i: int) -> int:   return x | (1 << i)
def clear_bit(x: int, i: int) -> int: return x & ~(1 << i)
def toggle_bit(x: int, i: int) -> int: return x ^ (1 << i)


Fumbling these four under pressure is the tell that bit manipulation isn't in your hands yet. They should come out without hesitation, the way you write a for-loop.

Three formulas that fall straight out of the same idea:

- Odd or even: `x & 1`
- Power of two: `n > 0 and (n & (n - 1)) == 0`
- Modulo 2^k: `n & ((1 << k) - 1)`

## The two low-bit idioms that show up everywhere

If you remember nothing else from a bit manipulation cheat sheet, remember these two lines. They look like incantations until you trace them once on paper.

**`x & -x` isolates the lowest set bit.**

In two's complement, `-x` is `~x + 1` — flip every bit, add one. That operation leaves the bits *above* the lowest set bit inverted and the bits *below* it (all zeros) untouched. AND-ing with `x` keeps only the lowest set bit.


x      = 0b0110_1100   (108)
-x     = 0b1001_0100
x & -x = 0b0000_0100   (4 — the lowest set bit, alone)


**`x & (x - 1)` clears the lowest set bit.**

Subtracting one flips the lowest set bit to 0 and turns every zero below it into a 1. AND-ing with the original leaves that region zeroed out.


x         = 0b0110_1100   (108)
x - 1     = 0b0110_1011
x & (x-1) = 0b0110_1000   (104 — lowest set bit gone)


This second one is Brian Kernighan's popcount loop, and it's the reason counting set bits can run in O(number of set bits) instead of O(bit width). It's also half of the power-of-two check above.

## XOR: four identities, three classic problems

XOR does more interview work than any other operator, and its entire personality fits in four identities.

1. `a ^ a == 0` — self-cancellation
2. `a ^ 0 == a` — identity
3. `a ^ b == b ^ a` — commutative
4. `(a ^ b) ^ c == a ^ (b ^ c)` — associative

Combine 1, 3, and 4 and you get the lemma that solves an entire family of problems: **fold a collection with XOR and every value appearing an even number of times cancels to zero. Only the odd-occurrence values survive.** Order doesn't matter, grouping doesn't matter, pairs vanish.

**Finding the single unpaired value.** Every element appears twice except one. Fold the array with `^` and the survivor is your answer — O(n) time, O(1) space. The hash map solution is also O(n) but costs O(n) space, which is why interviewers accept the XOR version.

python
def single_number(nums: list[int]) -> int:
    result = 0
    for value in nums:
        result ^= value
    return result


**Finding the missing number in [0, n].** XOR all the indices *and* all the values. Every present value cancels with its matching index; the missing number is left standing because its index has no partner.

python
def missing_number(nums: list[int]) -> int:
    result = len(nums)
    for i, value in enumerate(nums):
        result ^= i ^ value
    return result


**Finding two unpaired values.** This is the senior-signal version. A single fold gives you `a ^ b` and nothing you can split. The move: `a ^ b` has a 1 in every position where `a` and `b` differ, so grab any such bit — `diff = xor_all & -xor_all` — and partition the whole array on it. Every paired value stays with its twin; `a` and `b` land on opposite sides. Fold one side, then get the other with `b = xor_all ^ a`.

python
def two_singletons(nums: list[int]) -> list[int]:
    xor_all = 0
    for value in nums:
        xor_all ^= value
    diff = xor_all & -xor_all      # any bit where the two differ
    a = 0
    for value in nums:
        if value & diff:
            a ^= value
    return [a, xor_all ^ a]


Trace it on `[1, 2, 1, 3, 2, 5]`: `xor_all` is 6, `diff` is 2, the bit-set group folds to 3, and `6 ^ 3` is 5.

Two more XOR one-liners worth carrying: swapping without a temporary variable (`a ^= b; b ^= a; a ^= b`), and Hamming distance, which is just `popcount(a ^ b)`.

## Counting set bits three ways

"How many 1s?" is the popcount question, and interviewers want the answers in a particular order.

python
def popcount_builtin(x: int) -> int:
    return x.bit_count()          # Python 3.10+, O(1) on fixed-width ints

def popcount_str(x: int) -> int:
    return bin(x).count("1")

def popcount_kernighan(x: int) -> int:
    count = 0
    while x:
        x &= x - 1                # clear the lowest set bit
        count += 1
    return count


Say the idiomatic one-liner first, then offer the Kernighan loop as the language-agnostic version that shows you understand *why* it's fast. That sequencing — library answer, then from-scratch defense — reads as fluency rather than memorization.

The naive alternative loops over all 32 bits and checks each one. It's correct, it's fine, and it's the version that tells an interviewer you haven't internalized `x & (x-1)` yet.

## Bitmasks: subsets, state, and where DP enters

A bitmask uses an integer as a set: bit `i` set means element `i` is in the set. That gives you O(1) membership tests, unions with `|`, intersections with `&`, and complements with `& ~mask`.

Enumerating every subset of an n-element array is then a single loop from `0` to `2^n - 1`:

python
def all_subsets(nums: list[int]) -> list[list[int]]:
    n = len(nums)
    return [
        [nums[i] for i in range(n) if mask & (1 << i)]
        for mask in range(1 << n)
    ]


Where this gets serious is bitmask DP, where the mask *is* the state. The classic shape is the travelling salesman formulation: `dp[mask][i]` is the cheapest way to visit the set `mask` and end at node `i`. The state space explodes as `2^n * n`, which is exactly why the technique only shows up with small constraints — if you see `n <= 20` on a hard problem, bitmask DP is a strong hypothesis to test.

Bitmasks also show up disguised as "track flags without a set," permission systems in systems-flavored interviews, and any problem that hands you a `visited` array and a memory constraint.

## Other tricks worth knowing once the basics are in your hands

These aren't foundational, but each one appears often enough to be worth a line in your notes.

| Trick | Code | Notes |
| --- | --- | --- |
| Swap without temp | `a ^= b; b ^= a; a ^= b` | Compilers often do it better anyway |
| Fast exponentiation | shift `n` right, square `x` each step | O(log n) instead of O(n) |
| Gray code | `g = i ^ (i >> 1)` | Adjacent values differ by one bit |
| Add without `+` | XOR for sum, AND-shift for carry, repeat | Loop until carry is zero |
| Reverse bits | Shift the result left, OR in `x & 1`, shift `x` right | 32 iterations |
| Maximum XOR pair | Bitwise trie | Greedy from the top bit down |
| Clear a range of bits | `x & ~((1 << (k+1)) - 1)` | Careful with the off-by-one |

Add without `+` is the one people ask about, and it's short:

python
def add(a: int, b: int) -> int:
    while b:
        carry = a & b
        a ^= b
        b = carry << 1
    return a


XOR adds without carrying; AND identifies where a carry is owed; shifting the carry left puts it in the right column. Repeat until there's nothing left to carry.

## Which trick goes with which problem

This is the section a good bit manipulation cheat sheet has that most don't: the mapping from problem to technique, so you can recognize the pattern instead of guessing.

| Problem | Number | Technique |
| --- | --- | --- |
| Single Number | 136 | XOR fold |
| Single Number II | 137 | Per-bit counting / carry simulation |
| Single Number III | 260 | XOR fold + partition on lowest set bit |
| Missing Number | 268 | XOR indices against values |
| Number of 1 Bits | 191 | Popcount |
| Counting Bits | 338 | DP over popcount |
| Power of Two | 231 | `n & (n-1) == 0` |
| Power of Four | 342 | Power-of-two check + bit position |
| Reverse Bits | 190 | Shift-and-build |
| Hamming Distance | 461 | `popcount(a ^ b)` |
| Sum of Two Integers | 371 | XOR + AND-carry arithmetic |
| Subsets | 78 | Bitmask enumeration |
| Maximum XOR of Two Numbers | 421 | Bitwise trie, greedy from the top bit |

Five tricks covering roughly fifteen problems. That ratio is the whole argument for learning bit manipulation as patterns rather than as trivia.

## Practicing a cheat sheet is different from reading one

Here's the awkward part: you can read all of the above and still freeze when a bitwise question appears, because the failure mode isn't ignorance — it's retrieval. Under time pressure, you need to know *which* of the five buckets a problem belongs to before you start writing, and that recognition only comes from reps.

[👉 AlgoMonster](https://algo.monster/subscribe?fpr=li69) is built around that gap. It's a pattern-based interview prep platform from ex-Google engineers and competitive programmers, and bit manipulation is treated as one of the curriculum's patterns rather than as a standalone topic. The structure is text-and-diagram based, which suits bit problems — you can't really watch someone explain two's complement; you need to see the bits change.

What's on the platform, per its own pricing page:

- 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials
- All patterns, lessons, and problems unlocked with a Pro plan
- Unlimited AI assistant usage and unlimited Speedrun (timed practice) sessions
- Real interview questions from top tech companies, refreshed monthly
- Decision flowcharts for working out which pattern a problem needs — a cheat sheet for pattern selection, which is the skill the table above is trying to build
- Bonus courses included: CS fundamentals and a "build an LLM from scratch" course
- Member discounts on live bootcamps

Two honest limitations. There are no video walkthroughs at all — if you learn by watching someone code, this isn't the format for you. And the problem count is deliberately small compared to LeetCode's thousands, so most people pair it with a larger question bank for volume. AlgoMonster teaches the framework; LeetCode builds the speed.

## AlgoMonster pricing: every plan on the site

There's no monthly tier and no team plan. Three options:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of the curriculum — enough to see whether the text-and-diagram format works for you | $0 | — | [ Start on the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything in the Pro tier for 12 months: all patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, monthly-updated company questions, bonus courses, bootcamp discount | $99/year (about $8.25/month) | Billed annually | [ See the Pro Yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates, paid once | $189 | Billed once | [ Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

One caveat worth keeping in mind: AlgoMonster runs promotions often, and the prices on the subscribe page have moved around over time — reviews from even a few months ago quote numbers that don't match today's checkout. Confirm the current figure on the page rather than trusting a cached number, including the ones in this article.

On the annual-versus-lifetime question, the math is simple enough. If you're prepping for one interview loop over the next couple of months, the yearly plan is the lower-commitment option. If you expect to change jobs again — which most engineers do every few years, and each move means re-prepping — the lifetime plan costs less than two years of annual renewals. The pattern knowledge doesn't rot; that's the asset you're actually buying.

## A few quick answers

**Do I need to memorize all of this?** No. Memorize the four masks, `x & -x`, `x & (x-1)`, and the four XOR identities. Everything else in this cheat sheet is derivable from those.

**Which language should I practice in?** The one you'll interview in. Python has `int.bit_count()` and unbounded integers; Java and C++ have fixed-width ints and real overflow, which changes a few answers. Both are fine — just know which world you're in.

**Is bit manipulation actually common in interviews?** It's rarely the centerpiece of a loop, but it's a frequent curveball: the warm-up that hides an O(1)-space trick, the follow-up after you finish early, the systems question at an infrastructure team. It's also the topic where a candidate who understands binary looks obviously stronger than one who memorized a formula.

**What's the fastest way to get from reading this to using it?** Pick the three problems that map to XOR folding, solve them from scratch without looking at the code above, then come back and check. Reproduction is the test; recognition under a timer is the real one.
