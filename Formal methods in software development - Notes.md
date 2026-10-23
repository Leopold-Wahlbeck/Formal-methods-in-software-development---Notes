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

**Questions you can use this:**
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

# 4. Hoare Logic

**Questions you can use this:**
* Prove a triple that has a while loop, using an invariant
* Partial vs total correctness
* What ⊢ { true } C { false } means
* ⊢ vs ⊨ and soundness 
* The faulty assignment rule vs Floyd's rule
* Derive a rule for a new construct (repeat ... until)

## 4.1 Hoare triples

**⊢ { P } C { Q }** where:
* P = precondition, 
* C = code, 
* Q = postcondition.

The code uses: `skip`, `v := E`, `C1; C2`, `if G then C1 else C2 fi`, `while G do C od`.

| Notation      | Name                | Meaning                                                                    |
| ------------- | ------------------- | -------------------------------------------------------------------------- |
| { P } C { Q } | partial correctness | If P holds at the start **and C terminates**, then Q holds at the end      |
| [ P ] C [ Q ] | total correctness   | If P holds at the start, then C **does terminate**, and Q holds at the end |
**Total = partial + termination**. A program that never terminates satisfies *every* partial triple. 

Semantics (⊨), as on the slides:
- ⊨ { P } C { Q } = ∀s₁ s₂. P(s₁) ∧ (C, s₁) ⇓ s₂ ⇒ Q(s₂)
- ⊨ [ P ] C [ Q ] = ∀s₁. P(s₁) ⇒ ∃s₂. (C, s₁) ⇓ s₂ ∧ Q(s₂)

(C, s₁) ⇓ s₂ means “C, started in state s₁, finishes in state s₂”.

## 4.2 The rules


|Rule|If you have… (above the line)|…then you get (below the line)|
|---|---|---|
|Skip|nothing needed|⊢ { P } skip { P }|
|Assignment|nothing needed|⊢ { Q[x\E] } x := E { Q }|
|Sequence|⊢ { P } C1 { M } and ⊢ { M } C2 { Q }|⊢ { P } C1; C2 { Q }|
|If|⊢ { P ∧ G } C1 { Q } and ⊢ { P ∧ ¬G } C2 { Q }|⊢ { P } if G then C1 else C2 fi { Q }|
|Consequence|P ⇒ P′ and ⊢ { P′ } C { Q′ } and Q′ ⇒ Q|⊢ { P } C { Q }|
|While (partial)|⊢ { P ∧ G } C { P }|⊢ { P } while G do C od { P ∧ ¬G } (P = the invariant)|
|While (total)|< is well-founded and ∀v₀. ⊢ [ P ∧ G ∧ E = v₀ ] C [ P ∧ E < v₀ ]|⊢ [ P ] while G do C od [ P ∧ ¬G ] (E = the variant)|

**Derived rules (use these in proofs)**

|Rule|If you have… (above the line)|…then you get (below the line)|
|---|---|---|
|While (derived)|P ⇒ I and ⊢ { I ∧ G } C { I } and I ∧ ¬G ⇒ Q|⊢ { P } while G do C od { Q }|
|Assignment (derived)|P ⇒ Q[v\E]|⊢ { P } v := E { Q }|
|Sequence + assignment|⊢ { P } C { Q[v\E] }|⊢ { P } C; v := E { Q }|
|Repeat (Oct 2024)|P ⇒ I and ⊢ { I } C { I } and I ∧ G ⇒ Q|⊢ { P } repeat C until G { Q }|

## Assignment works backwards

**Q[x\E]** means “Q with every x replaced by E”.
* **Example**: ⊢ { true } x := 5 { x > 0 }. Here (x > 0)[x\5] = 5 > 0 = true ✓

**Several assignments in a row:** substitute starting from the **last** one and work towards the first. 

```
{ ? }  x := x − 1;  y := y + 1  { x + y = n }

after  y := y + 1:   (x + y = n)[y\y+1]  =  x + (y + 1) = n
after  x := x − 1:   [x\x−1]             =  (x − 1) + (y + 1) = n
                                         =  x + y = n
```

## 4.4 The derived while rule = 3 checks

![[Pasted image 20261023120229.png]]

The invariant I is something that is **true every time you are at the loop head:** before the first iteration, after every iteration, and when the loop exits.
1. ${}P\implies I{}$: the invariant holds when you first reach the loop.
2. ⊢ { ${}I{}$ ∧ ${}G{}$ } ${}C{}$ { ${}I{}$ }: if ${}I{}$ holds and the guard is true, then after one run of the body, ${}I{}$ still holds. 
3. ${}I{}$ ∧ ${}\neg G \implies Q{}$ : when the loop stops (the guard is false), ${}I{}$ together with ${}\neg G{}$ gives you the postcondition.   

## 4.5 How to find the invariant
1. Make a **small table of values** for a few iterations and look for something that **never changes**. Often its "one variable goes down while another goes up", which gives a sum that always stays constant. 
2. Look at the goal: ${}I ∧ \neg G{}$ **must give Q**. Pick I so that the exit condition "pins down" the answer.
3. Add the **bounds** you need at the exit (like ${}x\geq 0{}$), so that ${}\neg G{}$ gives you an exact value. 

##### Worked example
```
⊢ { x = n ∧ x ≥ 0 } 
y := 0; 
while x > 0 
	do 
		x := x − 1; 
		y := y + 1 
	od 
{ y = n }
```

**Table (n=3):** 
${}(x,y) = (3,0) \to (2,1) \to (1,2) \to (0,3){}$. 

**Invariant**: ${}I ≡ x + y = n ∧ x ≥ 0{}$

**Proof:**
* **y := 0** (derived assign rule): we need ${}x = n ∧ x ≥ 0 \implies I{}$[y\0], i.e. x + 0 = n ∧ x ≥ 0 ✓. So ⊢ { x = n ∧ x ≥ 0 } y := 0 { I }.
* **While loop**: (derived while rule), with precondition ${}I{}$:
	1. ${}I \implies I \space (✓){}$
	2.  ⊢ { ${}I ∧ x > 0{}$ } ${}x := x − 1; y := y + 1{}$ { ${}I{}$ }. Substituting backwards, it’s enough to show ${}I ∧ x > 0 \implies (x − 1) + (y + 1) = n ∧ x − 1 ≥ 0{}$. This holds: the sum is still x + y = n, and x > 0 gives x − 1 ≥ 0 ✓
	3. ${}I ∧ ¬(x > 0) \implies y = n:{}$ we have ${}x \geq 0{}$ and ${}x \leq 0{}$, so ${}x=0{}$, and then ${}0 + y = n \space ✓{}$ 
* Combine the two parts with the SEQ rule. 
**Why x ≥ 0 has to be in ${}I{}$:** without it, (3.) fails. ¬(x > 0) only gives x ≤ 0, not x = 0.

## 4.7 Faulty assignment rule
The “forwards” rule ⊢ { P } x := E { P ∧ x = E } is **unsound**.

**Counter example:** take P ≡ x = 0 and E ≡ x + 1. The rule gives  
{ x = 0 } x := x + 1 { x = 0 ∧ x = x + 1 }.  
The postcondition is **false** (x = x + 1 is impossible), but the program obviously terminates with x = 1. That’s nonsense.

**Floyd’s rule** (sound): ⊢ { P } x := E { ∃v. (x = E[x\v]) ∧ P[x\v] }, where v = the **old** value of x.  
Same example: { x = 0 } x := x + 1 { ∃v. x = v + 1 ∧ v = 0 }, which simplifies to x = 1 ✓ (correct).

## 4.8 Total correctness needs a variant

	< is well-founded    ∀v₀. ⊢ [ P ∧ G ∧ E = v₀ ] C [ P ∧ E < v₀ ]
	──────────────────────────────────────────────────────────────
		          ⊢ [ P ] while G do C od [ P ∧ ¬G ]
  
- **Variant E** = a value that **strictly decreases** on every iteration and can’t decrease forever. Example: n, using < on the natural numbers.
- **Well-founded** = there is no infinite descending chain.
- **Example:** `while n ≠ 1 do n := n − 2 od`
    - Partial { true } C { n = 1 } holds with ${}I{}$ = true, because the loop can only exit when n = 1.
    - Total does **not** hold: n = 0 gives 0, −2, −4, … forever.
    - It _is_ total with P ≡ odd(n) ∧ n ≥ 1, using P as the invariant and n as the variant.
# 5. Dafny & invariants

**Ex:** you get a Dafny method that works on an array, and you have to:
* a) write the spec: the ***requires***, ***ensures*** or a ***predicate***
* b) write the loop body with its ***invariant***


## 5.1 Weakest preconditions (wp): how tools like Dafny use Hoare logic

**wp(C,Q)** = the weakest precondition that makes *⊢ { wp(C, Q) } C { Q }* true. Any other valid precondition P implies it. 
${}\implies{}$ To prove ⊢ { P } C { Q }, it is enough to prove P ⇒ wp(C, Q).
```
wp(skip, Q)                      = Q
wp(v := E, Q)                    = Q[v\E]
wp(C1; C2, Q)                    = wp(C1, wp(C2, Q))
wp(if G then C1 else C2 fi, Q)   = (G ⇒ wp(C1, Q)) ∧ (¬G ⇒ wp(C2, Q))
```

**There is no equation for while**. You have to supply an invariant, and finding invariant can't be automated in general. That's why Dafny makes you write `invariant`.

**The VCs for** `init; while G do body od` **with invariant Inv**:
1. ${}P \implies wp(init, Inv){}$
2. ${}Inv ∧ G \implies wp(body, Inv){}$
3. ${}Inv ∧ \neg{G} \implies Q{}$
These are the same three checks as the while rule: enter, keep, exit.

**Revision lecture example:** ⊢ { n ≥ 0 } z := 0; while z < n do z := z + 1 od { z = n }
1. n ≥ 0 ⇒ Inv[z\0]
2. Inv ∧ z < n ⇒ Inv[z\z+1]
3. Inv ∧ ¬(z < n) ⇒ z = n
Take **Inv ≡ z ≤ n**: (1) 0 ≤ n ✓ (2) z < n ⇒ z + 1 ≤ n ✓ (3) z ≤ n ∧ z ≥ n ⇒ z = n ✓
**Careful with the order for several assignments:** for `k := 0; a := 0; b := 1`, wp gives ${}Inv[b\1][a\0][k\0]{}$. The **last** assignment is substituted first.

## 5.2 Dafny cheat sheet

```
method name(a: array<int>, x: int) returns (r: int)
  requires P                     // precondition
  ensures  Q                     // postcondition
  modifies a                     // only if the method WRITES to a
{
  var i := 0;
  while i < a.Length
    invariant 0 <= i <= a.Length
    invariant forall k :: 0 <= k < i ==> ...
    decreases a.Length - i       // the variant (Dafny usually guesses it)
  {
    ...
    i := i + 1;
  }
}

predicate sorted(a: array<int>)
  reads a                        // needed when a predicate reads an array
{
  forall i, j :: 0 <= i <= j < a.Length ==> a[i] <= a[j]
}
```

* `==` means equal, `:=` assigns, `==>` means implies, and `&&`, `||`, `!` work as usual.
* **`forall k :: range ==> property`** but **`exists k :: range && property`**. Mixing these up is a classic mistake.
* Indices go from 0 to a.Length -1. `new int[n]` creates an array. `return x;` exits the method. 
* `for i := 0 to a.Length {...}` loops i = 0, 1, ..., a.Length - 1. 

## 5.3 Some spec patterns

|English|Dafny|
|---|---|
|j is the index of the largest element|`0 <= j < a.Length && forall i :: 0 <= i < a.Length ==> a[i] <= a[j]`|
|a is sorted|`forall i, j :: 0 <= i <= j < a.Length ==> a[i] <= a[j]`|
|b is a reversed|`forall k :: 0 <= k < b.Length ==> b[k] == a[a.Length - k - 1]`|
|a is a palindrome|`forall i :: 0 <= i < a.Length ==> a[i] == a[a.Length - 1 - i]`|
|key is found at r|`0 <= r < a.Length && a[r] == key`|
|key is not in a|`forall i :: 0 <= i < a.Length ==> a[i] != key`|

## 5.4 The invariant recipe for array loops

**The invariant = the poistcondition, but only for the part already processed.** Replace `a.Length` by `i`. Then add the bounds `0 <= i <= a.Length`.

![[Pasted image 20261023151245.png]]

## 5.5 Worked example: find_max
```
method find_max(a: array<int>) returns (j: int)
  requires a.Length > 0
  ensures 0 <= j < a.Length
  ensures forall k :: 0 <= k < a.Length ==> a[k] <= a[j]
{
  j := 0;
  var i := 1;
  while i < a.Length
    invariant 0 <= j < a.Length
    invariant 1 <= i <= a.Length
    invariant forall k :: 0 <= k < i ==> a[k] <= a[j]   // postcondition with a.Length → i
  {
    if a[j] < a[i] { j := i; }
    i := i + 1;
  }
}
```


## 5.6 Two more patterns

```
invariant 0 <= lo <= hi <= a.Length
invariant forall i :: 0 <= i < lo ==> a[i] < key           // left part: too small
invariant forall i :: hi <= i < a.Length ==> key < a[i]    // right part: too large
```

At exit `lo == hi`, so every index is in one of the two parts, and therefore the key is nowhere in the array. 

**Returning early (palindrome)**: check pairs with a loop and `return false` as soon as one fails. The invariant says "all pairs checked so far match": 

```
invariant 0 <= i <= a.Length / 2
invariant forall k :: 0 <= k < i ==> a[k] == a[a.Length - 1 - k]
```


# 6. Separation logic

#### Questions like:
* Prove a short heap program step by step and find Q
* Find the error in a wrong "proof"
* Explain why a ↦ x ∗ a ↦ y is false, and whether p ∗ p can ever be true
* VeriFast: heap chunks, open / close
* "Too much separation" (trees with shared parts)

## 6.1 Why separation logic? The frame problem

In Hoare logic you have to keep repeating everything that **didn't** change, and with pointers you also have to say they don't point to the same place (${}a \neq b{}$). Separation logic fixes this with * and the **frame rule**, which gives you **local reasoning:** you only talk about the memory the code actually touches. 

**State = (s,h):** a **store** s (${}variables \to values{}$) plus a **heap** h (${}adresses \to values{}$).

Example heap: [3 ↦ 4, 4 ↦ 29, 29 ↦ 0]

## 6.2 Assertions
| Assertion         | Meaning                                                                                |
| ----------------- | -------------------------------------------------------------------------------------- |
| **emp**           | the heap is empty                                                                      |
| **e₁ ↦ e₂**       | the heap has exactly one cell: address e₁ contains e₂                                  |
| **p ∗ q**         | the heap can be split into two **disjoint** parts, p holds for one and q for the other |
| **⟨b⟩**           | b is true about the variables, and the heap is empty                                   |
| **v ↦ E₁, …, Eₙ** | shorthand for v ↦ E₁ ∗ v+1 ↦ E₂ ∗ … (n cells in a row)                                 |
| **_**             | “some value” we don’t care about, e.g. a ↦ _                                           |
**Rules to know:** emp ∗ p = p · p ∗ q = q ∗ p · (p ∗ q) ∗ r = p ∗ (q ∗ r) · ⟨true⟩ = emp · ⟨b₁⟩ ∗ ⟨b₂⟩ = ⟨b₁ ∧ b₂⟩

## 6.3 New commands and their rules

```
[E1] := E2               write E2 into the heap at address E1
v := [E]                 read the heap at address E into v
v := cons(E1,…,En)       allocate n new cells
dispose(E)               free the cell at E

UPDATE    ⊢sep { E ↦ _ } [E] := E′ { E ↦ E′ }
CONS      ⊢sep { emp } v := cons(E1,…,En) { v ↦ E1,…,En }      (v not free in E1…En)
DISPOSE   ⊢sep { E ↦ _ } dispose(E) { emp }
READ      ⊢sep { E ↦ v″ } v := [E] { ⟨v = v″⟩ ∗ E ↦ v″ }        (v not free in E)

FRAME            ⊢sep { P } C { Q }
            ─────────────────────────────    when NO variable free in R
            ⊢sep { P ∗ R } C { Q ∗ R }         is modified by C
```

**How to use the frame rule:** prove each command on just the cell it touches, then "frame on" the rest of the heap as R. **Always check the side condition:** R must not mention a variable that the command assigns to. 

## 6.4 Worked example

`⊢sep { emp } x := cons(1); y := cons(2); [x] := y; z := [y]; dispose(y) { Q }`

Heres what the heap looks like after each command:

![[Pasted image 20261023155323.png]]


## 6.5 Some facts

> **Why is a ↦ x ∗ a ↦ y always false?** ∗ requires splitting the heap into two **disjoint** parts, and each of them would have to contain address a. A cell can only be in one part, so no heap satisfies it.

> **Can p ∗ p ever be true?** Yes, but only if p needs no heap cells. For example emp ∗ emp = emp, and ⟨b⟩ ∗ ⟨b⟩ = ⟨b⟩. If p claims any cell, the two copies would need that cell twice.

> **Find the error in the proof:** look at the **frame rule**. They framed R ≡ x ↦ y around `y := cons(x)`, but that command modifies y, which appears in R. That breaks the side condition.

> **VeriFast:** VeriFast pushes a collection of separation logic assertions, called **heap chunks** (joined by ∗), symbolically forward through the code. `//@ open` unfolds a predicate chunk into its parts, and `//@ close` packs the parts back into the predicate.


## 6.6 Predicates for data structures
```
list(a, [])      = emp ∗ ⟨a = null⟩
list(a, x · xs)  = ∃b. a ↦ x, b ∗ list(b, xs) ∗ ⟨a ≠ null⟩
```

Each node is 2 cells (value, next), and the rest of the list is **separate from it**.

"**Too much separation**": a tree predicate demands that the subtrees are disjoint. So the code that shares a subtree, like `b := cons(null, 1, null); a := cons(b, 2, b)`, can never be described as a tree(a, ...). 


