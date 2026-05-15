## What breaks if the cache is mutable instead of write-once?

The main thing that breaks is you can't trust the cache anymore.

Say the LLM gets one wrong and labels a vendor as "Travel" when it should be "Meals & Entertainment." With write-once, that bad entry is stuck there but at least it is consistent. Every time that vendor shows up, you get the same wrong answer, and you can find it and fix it. With mutable, a correct entry could get overwritten later by a wrong one, and you would never even know it happened.

Then there is the concurrency thing. If two transactions from the same vendor come in at the same time and both hit the LLM, the LLM might give different answers for each. Whichever one saves last is what stays in the cache. So now you have two transactions from the same vendor with different categories in your books. That is a mess.

And then auditing becomes impossible. If someone asks you "why did this get labeled Travel?" you need a real answer. If the cache keeps changing, the answer is just "because that is what it said at the time" which does not work when you are doing accounting.

Write-once keeps things predictable. Mutable makes the whole system unreliable.
