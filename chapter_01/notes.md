## Chapter 1 - Just a... whirl of wind
### (The Semantics of TLA+ and PlusCal)
  

Basically, we're going to start writing up an algorithm to send money from one bank account to another.
We'll leave in a flaw so that TLA+ has something to do.

A few spec notes:
- The name of a TLA+ module _needs_ to match the filename. Otherwise, TLA+ will consider the spec invalid.
- Anything above the dashes (`----`) and below the equals signs (`====`) are ignored
- Single line comments: `\*`; block comments: `(* *)`
- PlusCal is technically written inside a comment
    - I guess there's something that goes through and transpiles it to TLA+?
- Imports are done via the `EXTENDS` keyword
    - Since we're doing something "algorithmic"(?), we'll use `EXTENDS Integers`. This seems similar to Coq IIRC. 
    Something to do with importing various proven things about Integer behavior maybe? Who knows


#### Specifying
For our little demo, there are two things we want to keep track of:
- The set of people with accounts
- How much money those people have

```
EXTENDS Integers

(*--algorithm wire
    variables
        people = {"alice", "bob"},
        acc = [p \in people |-> 5];
begin;
    skip;
end algorithm;
*)
```

We'll start by defining the people in our system. In this case, `people` is an unordered set of strings (human names).
We also define a function, `acc`. Functions in TLA+ are maybe more like list or dictionary comprehensions in Python.  
You can kind of think of `[p \in people |-> 5]` as `{p: 5 for p in people}`. 
A (slightly) less trivial example would be something like `[x \in 1..10 |-> 2*x]`, which would translate to `[2 * x for x in range(1,11)]`

With the key players of our scenario defined, we can now define the relveant variables for an action we'd like to test. 
We'll start with just a single transaction, from alice to bob, for 3 currency units.  
Additionally, we'll define an _Invariant_: a property we want to remain true at every step.
The invariant states that everyones bank account should have 0 or more currency units.
The TLA+ model checker (not sure if this is the same as TLC or not), will make sure this is true at every step.

#### Implementing
OK, we've specified some variables and an invariant.
Now we'll actually write the procedure of _how_ currency is moved from one person to another.  

_Language note:_ If you're assigning to a variable for the first time, use `=`. If you're reassigning, use `:=`.

All of this, plus the new variables we defined earlier, look like this:
```
(*--algorithm wire
    variables
        people = {"alice", "bob"},
        acc = [p \in people |-> 5],
        sender = "alice",
        receiver = "bob",
        amount = 3;
define
    NoOverdrafts == \A p \in people: acc[p] >= 0
end define;

begin;
    Withdraw:
        acc[sender] := acc[sender] - amount;
    Deposit:
        acc[receiver] := acc[receiver] + amount;
    skip;
end algorithm;
*)
```

Also important to note is that `Withdraw` and `Deposit` are _labels_.
Everything under one label can be thought of as an atomic operation.
So, if we had multiple statements we could kind of assume that they're happening inside a lock or something.

#### Verifying
Alright, time to verify what we've written is correct so far. To do that, we'll create a `model`. 
This is done via the section at `TLC Model Checker` -> `New Model`. 
We'll also need to manually add the invarient we've written.

Since there's only one starting state (and, subsequently, one path for our logic to follow), TLC only executed one set of instructions:
1. Choose an initial state (we only had one possible initial state)
2. Make sure that state didn't violate any of our invariants
3. Execute `Withdraw`
4. Check that the state of the system didn't violate our invariant(s)
5. Execute `Deposit`
6. Check that the state of the system didn't violate our invariant(s)

That's it

#### Initial Conditions
_Language Note_: PlusCal (And TLA+) provides the `a..b` notation to mean "all integers between a and b (inclusive)"

By changing the amount of currency from `3` to, say, `1..6`, we can have TLA+ check our spec for all inputs in that range.
Unsuprisingly, since we give everyone 5 units to start, TLC will show us that our `NoOverdrafts` invariant is violated when trying to withdraw 6 from Alice's account.
Chaning the range of `amount` to be capped at `acc[sender]` fixes this problem.

#### Multiple Processes
_Note to self: you need to run `toolbox` as sudo_

What I think is the last piece of our "wire transfer" program: simultaneous transfers. 
In PlusCal, each algorithm happening simultaneously belongs to its own `process`. 
Processes can have their own local variables and code. 
It sounds somewhat similar to the notion of an OS process. Maybe that's the point.  
Each thing can have its own local state, and that state may or may not be accessed by another process (I should probably clean up this definition at some point).

In the case of running simultaneous transfers, each process is just running the same code:
```
process Wire \in 1..2
    variables
        sender = "alice",
        receiver = "bob",
        amount \in 1..acc[sender];
...
end process;
end algorithm;
```

This fails because, in this scenario, we're saying it's OK for Alice to send all of their money twice at one moment in time. This invalidates our invariant, and so the model checker tells us so.  

Even if we change the process to check whether funds are available, the model checker fails.

