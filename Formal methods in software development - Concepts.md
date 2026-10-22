
### Interleaving
**Interleaving** = how the steps of several processes get **mixed together** into one single sequence.

Think of it like **shuffling two decks of cards** together:

- Each process is one deck, and its own cards always stay **in order**.
- But the cards from the two decks can be mixed in **any** way.

Promela runs processes “on one processor”: at each step the scheduler picks **one** process and runs **one** statement of it. Which process goes next is random, so every mix is possible.