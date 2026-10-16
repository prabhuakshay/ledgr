# Drafts are kept apart from Transactions

A Draft is stored as its own record, not as a Transaction with a draft status, and only becomes a Transaction when the user posts it. Every Balance, Net Worth and report query reads Transactions and Splits directly; a status flag would force each of them to filter drafts out, and one forgotten filter would silently show wrong numbers. Keeping Drafts apart means a Draft can never move a Balance, at the cost of a second, smaller shape that has to be validated again and copied into a Transaction at post time.

Note: a Transaction still cannot be dated after today, but a Draft can. Posting such a Draft early records the Transaction on today's date rather than refusing it, because the user posting it is saying the money has moved now.

Posting is all or nothing: posting several Drafts at once is refused unless every one of them is complete, not an estimate, and dated today or earlier, so no Draft is ever skipped silently. Posting a future Draft on its own says which date it will be recorded on before the user confirms.
