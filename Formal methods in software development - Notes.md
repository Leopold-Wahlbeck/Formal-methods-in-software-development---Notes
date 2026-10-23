# 1. Promela & SPIN
Explaining rendezvous vs buffered channels ${}-{}{}$ explaining ${}\text{atomic} -{}$listing possible final values of variables${}-{}$explaining deadlock + end labels ${}-{}$ writing a small process.    
##  1.1 The big idea
* Promela = a small language for *modelling* concurrent systems. It is not meant for writing real programs. 
* **SPIN** = the model checker. It is built to *prove you wrong*, so it searches for a **counter example.**
* The search is **exhaustive**: it resolves the non-determinism in every possible way. 
	* It finds nothing ${}\to{}$ ${}\text{ERRORS: 0}{}$ ${}\to{}$ the property is verified. 
	* It finds something ${}\to{}$ it saves the failing run in the "trail".
* There are two sources of non-determinism:
	1. Overlapping guards in ${}{if}{}$/${}do{}$ 
	2. The scheduling of processes (**interleaving**)

## 1.2 Syntax

int x = 0;                  // global = shared by all processes
active proctype P() {       // declares P and starts 1 copy
	  int i = 0;                // local to P
	  do
	  :: i < 3 -> i = i + 1     // :: guard -> statements
	  :: else  -> break         // else = only if all other guards are false
	  od
}

- `;` separates statements. `->` means exactly the same as `;`.
- `if` / `do`: if several guards are true, any one of them may be picked (SPIN tries all of them). **If no guard is true, the statement blocks.**
- `do` repeats forever. Only `break` (or `goto`) exits the loop.

## 1.3 Executability

EXECUTABILITY (when can a statement run?)

| Statement                                | Executable when…                                   |
| ---------------------------------------- | -------------------------------------------------- |
| assignment, `assert`, `printf`           | always                                             |
| expression statement, e.g. `(x == 1);`   | only when it is true, (wait here until (x == 1))   |
| `if` / `do`                              | when some guard is executable                      |
| `atomic { … }`                           | when its first statement (the guard) is executable |
| rendezvous `[0]`: send `!` / receive `?` | when a partner is ready at the same time           |
| buffered `[n]`: send `c ! e`             | when the buffer is not full (n < cap)              |
| buffered `[n]`: receive `c ? v`          | when the buffer is not empty (n > 0)               |

* A statement that is not executable **blocks**, and the process waits. 
* At each step, the scheduler pick **one non-blocked process** (nondeterministically).  
* ${}\implies{}$Other processes can sneak in **between any two statements.**
## 1.4 ${}atomic{}$
* `atomic {s1; s2; s3}` is **weakly atomic**: once it is entered, no other process interleaves, *unless* a statement *inside it blocks*.
* It is only entered when `s1` (its guard) is executable. 
* `d_step` = strongly atomic and deterministic. You only need to know it exists.

#### Method: "what are the possible final values?"
1. Run each process alone in your head. 
2. Mark the places where each one can **block** (guards, !, ?).
3. Remember that other processes can interleave between statements (but not inside `atomic`)
4. Try the extreme schedules: "P runs as early as possible" vs "Q runs all the way first".
5. A process that is stuck forever still gives a final state. Say that its stuck.

##### Example:
int x = 0; int y = 0;
active proctype P() {
  atomic { (x == 1); x = x + 1; y = x; }
}
active proctype Q() {
  do
  :: x < 5 -> x = x + 1;
  :: else  -> break;
  od
}

P waits until `x == 1`, and only Q can make that happen. After that it’s a race, and there are two possible runs:

![[Pasted image 20261022160026.png]]

**Answer:** x = 5 always, because Q counts up to 5. y = 2 (run A) or y = 0 (run B). In run B, P is blocked forever. Remember that, because it’s a deadlock (see 1.6).


## 1.5 Channels

`chan c = [0] of { int };   // [0] = rendezvous channel
`chan d = [2] of { int };   // [2] = buffered channel, holds 2 messages
`c ! e      // send the value of e
`c ? v      // receive into variable v?

**Rendezvous `[0]`**
- No value is ever stored in the channel.
- `c ! e` is executable only when another process is ready to do `c ? v` at the same moment, and the other way around.
- Both statements execute **together, in one step**, and the value goes straight from sender to receiver. _(This is the only place where two processes execute at once.)_
- If no partner is ready, the process blocks.

**Buffered `[n]`**: a FIFO queue
- `c ! e` is executable when the buffer is **not full** (n < cap).
- `c ? v` is executable when the buffer is **not empty** (n > 0).
- It is asynchronous: the sender can run ahead of the receiver.

**Pattern matching:** `c ? EAST` (a constant, not a variable) is only executable if the message _is_ `EAST`.

## 1.6 Deadlock & end labels
* **End location** of a process = its textual end, or a label starts with `end` (e.g. `end: `)
* **Valid end state** = every process is at an end location.
* **Deadlock**: let NEL = the running processes that are **NOT** at a valid **E**nd **L**ocation. Deadlook${}\iff{}$NEL ${}\neq{}$ ${}\emptyset{}$ and all ${}p \in NEL{}$ are blocking. 
* SPIN checks for this be default. It reports "**invalid end state**" in verification mode and "**timeout**" in simulation mode. 
* If waiting forever is fine (e.g. a server), the fi is to put `end: ` before the loop. 
## 1.7 Two SPIN tricks
* assert(expr) ${}\to{}$ an arror if expr is false, and SPIN gives you the trail.
* Want SPIN to find a path to a goal? Put `assert(false)` at the goal. The counter example trail **is** the solution.

# 2. LTL, Linear Temporal Logic

## (4 types of questions in this course):
1. Does this formula hold on this trace? 
2. Is this formula valid? 
3. Write LTL for an English sentance
4. Safety vs liveness

## 2.1 Traces
* AP = the atomic propositions, e.g. AP = {p,q}
* A **trace** τ = ${}I_{0},I_{1},I_{2},\dots{}$ is *infinite* sequence. Each ${}I_{i}{}$, says which propositions are true at position i.
* Its written as sets: τ = {p},{p,q},{},... where {} means nothing is **true**. 
* A run that stops is made infinite by looping on its last state. 
* τ|ₖ = the suffix of τ that starts at position k.
- **T ⊨ φ** (the transition system satisfies φ) iff **every** trace of T satisfies φ.

## 2.2 The three connectives (for this course)
|Symbol|Name|Spin syntax|
|---|---|---|
|□φ|always|`[]`|
|♦φ|eventually|`<>`|
|φ U ψ|until|`U`|

**Semantics (as on the slides):**

- τ ⊨ p iff I₀(p) = T, i.e. **a plain p only looks at position 0!**
- τ ⊨ □φ iff τ|ₖ ⊨ φ for **all** k ≥ 0
- τ ⊨ ♦φ iff τ|ₖ ⊨ φ for **some** k ≥ 0
- τ ⊨ φ U ψ iff τ|ₖ ⊨ ψ for some k ≥ 0, and τ|ⱼ ⊨ φ for all 0 ≤ j < k

Two things to notice:

- “k ≥ 0” includes _now_. So □p → p is valid.
- U **demands** that ψ really happens at some point. “p forever, q never” does **not** satisfy p U q.

**Here's what each formula demands from a trace**:
![[Pasted image 20261022172201.png]]

## 2.3 Rules to memorise
- **Duals:** ¬□p ≡ ♦¬p and ¬♦p ≡ □¬p. To push a ¬ inside, flip □ ↔ ♦.
- ♦p ≡ true U p
- □□p ≡ □p and ♦♦p ≡ ♦p
- □(p ∧ q) ≡ □p ∧ □q. But □(p ∨ q) is **weaker** than □p ∨ □q.
- ♦(p ∨ q) ≡ ♦p ∨ ♦q. But ♦(p ∧ q) is **stronger** than ♦p ∧ ♦q.
- ♦□p → □♦p is **valid**. □♦p → ♦□p is **not** valid.
- false → anything is valid. Example: (□p ∧ □¬p) → q, because the left side can never be true.

## 2.4 Method: "is it valid?" (the most common question)
**Valid = true on _every_ trace.**

1. The formula usually has the shape A → B. Try to build a **counter example**: a trace where A is true and B is false.
2. Try the small standard traces:
    - p forever
    - ¬p forever
    - p once, then ¬p forever
    - p ¬p p ¬p … (alternating)
3. Found a counter example → **not valid** (write the trace down). Can’t find one → **valid** (give a one-line reason).

**Example 1:** □♦p → ♦□p. Try p ¬p p ¬p …: □♦p is true (p keeps coming back), but ♦□p is false (p never stays). → **Not valid.**

**Example 2:** ♦□p → □♦p. If p is true forever from some point on, then p is true at infinitely many positions. → **Valid.**

### 2.6 English ${}\to{}$LTL
| English                       | LTL        |
| ----------------------------- | ---------- |
| always p                      | □p         |
| never p                       | □¬p        |
| eventually p                  | ♦p         |
| infinitely often p            | □♦p        |
| from some point on, always p  | ♦□p        |
| whenever p, eventually q      | □(p → ♦q)  |
| p holds until q happens       | p U q      |
| after q has happened, never p | □(q → □¬p) |
### 2.7 Safety vs liveness

- **Safety** = “something bad never happens.” Example: □¬(P_in_CS ∧ Q_in_CS).  
    → Every violation has a **finite bad prefix** that can never be repaired later.
- **Liveness** = “something good happens eventually.” Example: ♦P_in_CS.  
    → Every finite prefix can still be extended into a good trace, so there is **no finite counter example**.

# 3. Büchi automata & model checking
**How this connects**: SPIN checks T ⊨ φ, i.e. "every trace of the Promela model satisfies the LTL formula" To do that it uses **Büchi automata**: automata that read infinite traces. 

Questions you can use this:
- define B = (Q, δ, Q₀, F) and say when it accepts a word
- draw an automaton for an ω-expression, or write the ω-expression for a drawn automaton
- generalised Büchi automata: what they are, how acceptance differs, why they’re used
- the 5 steps of LTL model checking, why φ is negated, where the counter example comes from
- draw the product T ⊗ B

### 3.1 Definition

#### B = (Q, δ, Q₀, F) over an alphabet Σ:
- **Q**: a finite, non-empty set of **locations**
- **δ ⊆ Q × Σ × Q**: the **transition relation**. (q, a, q′) means “from q, reading a, go to q′”.
- **Q₀ ⊆ Q**: a non-empty set of **initial** locations
- **F ⊆ Q**: a set of **accepting** locations

#### **Execution** (w, v):
- w = a₀ a₁ a₂ … ∈ Σ^ω is the word being read.
- v = q₀ q₁ q₂ … ∈ Q^ω is the list of locations visited.
- It must start in an initial location (q₀ ∈ Q₀) and follow δ: (qᵢ, aᵢ, qᵢ₊₁) ∈ δ for all i.

**Acceptance:** B accepts ${}w \iff{}$ **there exists** an execution (w, v) where some accepting location ${}f \in F{}$ appears **infinitely often** in v.

**Language:** Lω(B) = { w ∈ Σ^ω | B accepts w }

Things to remember: 
* It only reads **infinite** words (ω-words). Can it accept a finite word? **No.**
* "There exists" means it's non-deterministic: one good path is enough.
* If there is no transition for the next letter, that execution dies. 
* Non-deterministic Büchi automata are strictly more expressive than deterministic ones.

### 3.2 ω-expressions

| Expression | Meaning                              |
| ---------- | ------------------------------------ |
| ab         | a followed by b                      |
| a + b      | a or b                               |
| a*         | a finitely many times (0 is allowed) |
| a^ω        | a infinitely many times (forever)    |

**Drawing rule**: the part under ^ω becomes a **loop through an accepting location** (drawn as a double circle). The part in front of it is the path that leads there. 

![[Pasted image 20261022215456.png]]


### 3.3 Method: automaton ↔ ω-expression

**Automaton → expression:**

1. For each accepting location f, find (the way **to reach** f) and (the **loops back** to f).
2. That accepting location gives: reach · (loop)^ω
3. Join the parts with +.

**Expression ${}\to{}$ automaton**: build the prefix as a chain of location, then put the ω part as a loop on an accepting location. For a "+", make two branches from the start.

### 3.4 Generalised Büchi automata (GB)
* GB = (Q, δ, Q₀, F), the same as before except **${}F = {F_{1},\dots,F_{k}}{}$** **is a set of sets** of accepting locations
* **It accepts** w iff **for every** i, at least one location in ${}F_{i}{}$ is visited infinitely often.
* **The difference:** a normal B needs *some* accepting location infinitely often. 
* **The difference:** a normal B needs *some* accepting location infinitely often. A GB needs every acceptance set ${}F_{_{i}}{}$ hit infinitely often. 
* **Why use them:** it's much easier to translate LTL into a GB (one ${}F_{i}{}$ per U-subformula). The GB is then turned into a normal B by making one copy of the automaton per ${}F_{i}{}$.

### 3.5 LTL model checking: the 5 steps (is T ⊨ φ?)
1. Build a generalized Büchi automaton GB¬φ for the **negation** ¬φ.
2. Turn it into an equivalent **normal** Büchi automaton B¬φ (same language). 
3. Build the **product** ${}T ⊗ B\negφ{}$. This contains exactly the runs of T that ${}B\negφ{}$ accepts, i.e. the runs that **violate** φ. 
4. **Search** the product for a path from an initial node to a **cycle through an accepting node**.
5. Found ${}\to T {}$ ⊭ φ and that path is a **counter example**. Not found → **T ⊨ φ**.

### 3.6 The product ${}T ⊗ B {}$ (definitions from the slides)
* **Nodes**: pairs ⟨s, q⟩ (a state of T together with a location of B)
* **Arrows: ⟨s, q⟩ → ⟨s′, q′⟩ iff s → s′ in T **and** (q, L(s′), q′) ∈ δ  → **The automaton reads the label of the _target_ state.****
* **Initial nodes:** ⟨s₀, q⟩ where s₀ is initial in T and (q₀, L(s₀), q) ∈ δ for some q₀ ∈ Q₀. The automaton first reads s₀’s own label.
* **Accepting nodes:** ⟨s, q⟩ with q ∈ F


##### **Worked example: 

![[Pasted image 20261023105418.png]]

###### **B¬φ accepts “infinitely often {}”, i.e. ¬♦□p ≡ □♦¬p:**
![[Pasted image 20261023104807.png]]
###### How the product was built, step by step:
1. **Initial node**: ${}s_{0}{}$ has label {}, and from q0 reading {} goes to q1, so the initial node is ⟨s0, q1⟩. Its accepting because ${}q1 \in F{}$.
2. ⟨s0, q1⟩: T goes ${}s0 \to s1{}$, and the automaton reads ${}L(s1) = {p}{}$, so ${}q1 \to q_{0}{}$. That gives **⟨s1, q0⟩**.
3. ⟨s1, q0⟩: s1 → s2, reading {p} keeps q0, so we get **⟨s2, q0⟩**.
4. ⟨s2, q0⟩: s2 → s1, reading {p} keeps q0, so we’re back at ⟨s1, q0⟩. That’s a cycle, but it is **not accepting**.
5. The only accepting node is never visited again, so there is no accepting cycle and **T ⊨ ♦□p**.

**Recipe for “draw ${}T ⊗ B{}$”:** start from the initial node(s), and for every T-arrow s → s′ look up which q′ the automaton goes to when it reads L(s′). Only draw the nodes you can actually reach. Mark initial nodes with an arrow and accepting nodes with a double border.
