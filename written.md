## What breaks if the cache is mutable instead of write-once?

Three things go wrong.

First, race conditions. If two transactions for the same vendor hit tier 3 simultaneously and the LLM returns different categories (it is nondeterministic), whichever write lands last wins. Now you have the same vendor categorized differently depending on timing.

Second, feedback poisoning. If the LLM miscategorizes once, that wrong answer enters the cache. With write-once, the damage is contained. One bad entry, but it is stable. With mutable, a correct entry can get overwritten by a later incorrect one. Worse, if cached results feed back into the LLM as few-shot examples, bad entries propagate into future calls creating a degradation loop.

Third, auditability dies. Bookkeeping needs a paper trail. If the cache mutates, you cannot explain why transaction #47 was categorized as "Travel" when the cache now says that vendor is "Meals & Entertainment." Auditors need deterministic categorization. Write-once gives you a stable mapping you can always point back to.

Mutable caches turn a deterministic lookup into a source of nondeterminism. In accounting that is not a tradeoff, it is a bug.