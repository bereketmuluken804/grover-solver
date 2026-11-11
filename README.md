# Report: Solving a Deployment Configuration Problem with Grover's Algorithm

## Objective

The main aim of these two tasks was to see how a quantum search
algorithm can be applied to a constraint satisfaction problem. These are
problems where many choices must satisfy a set of rules, and many hard
problems of this type belong to the NP family, where checking an answer
is easy but finding one means searching. Grover's algorithm speeds up
that search. A classical search checks about N candidates, while Grover
needs only about √N oracle calls, which is a quadratic speedup.

The problem here is a small deployment sandbox with three features:
public API, authentication, and background worker. A valid configuration
needs at least one execution mode, authentication whenever the API is
on, and never the API and the worker together.

## Implementation

In Task A, the requirements were translated into Boolean logic, and the
oracle was built by hand. The key step was reducing the rules to two
mutually exclusive cases: with the API off, the worker must be on; and
with the API on, authentication must be on and the worker off. The
oracle computes these cases into helper qubits using only X, CX, and CCX
gates, combines them into a validity flag, and flips the phase of the
valid states through phase kickback on an ancilla held in the \|−⟩
state. Every helper qubit is then uncomputed by running the steps in
reverse, so the oracle changes nothing but the phase.

This was checked on all eight inputs: valid configurations received a
minus sign, and both the input bits and the helper qubits came back
unchanged. Grover search then ran on top of this oracle and returned a
valid configuration, which was verified with the classical checker.

Task B made the solver configurable. It accepts an extra Boolean policy
written with `&`, `|`, `^`, `~`, and parentheses, checks it with a safe
parser, and combines it with the original rules. The combined rule is
then passed to `PhaseOracleGate`, which generates the oracle
automatically. Three test policies covered the required cases: exactly
one solution (`P & A & ~W`), several solutions (`A`), and none
(`P & W`). The result for each was checked classically, and the
generated oracle was confirmed to mark exactly the expected
configurations. Invalid policies are rejected with a clear reason, and
an impossible policy is reported as having no solution.

## Findings

More Grover iterations are not always better. The success probability
rotates instead of rising steadily: with three valid configurations out
of eight, one iteration gives 84.4%, two give only 2.3%, and three give
99.0% only because the state rotates past the target and comes back
around.

Grover cannot prove that no solution exists. If a run finds nothing,
that may only be an unlucky sample, so only an exhaustive classical
check can prove impossibility.

The quadratic speedup cannot be demonstrated on a problem with only
eight configurations. Classical enumeration needs at most eight checks
and gives an exact answer, while each Grover oracle call is a multi-gate
circuit that also costs more to simulate. Classical enumeration is
simply better here. The √N advantage describes how the cost grows with
problem size, not how fast this particular run is.

## Design Decisions

The first decision was to use k = 1 instead of k = 3. Both work, since k
= 1 gives 84.4% and k = 3 gives 99.0%. Because k = 1 is the first peak
(the formula gives k ≈ 0.69), it uses the fewest oracle calls, while k =
3 costs three calls, which is more than the classical average of about
2.7 checks, and only works because of wraparound. The price is a 15.6%
chance of a failed run, which is acceptable because the result is always
verified and a retry is cheap.

The second decision was to count solutions classically before running
Grover. The number of solutions determines how many rounds to run, so it
was computed first with an exhaustive check, which also catches the
zero-solution case before any quantum run. The alternative was to run
Grover without knowing the count, either guessing the number of rounds
or trying increasing numbers of rounds and verifying each result. That
would work for large problems where counting is impractical, but here it
would make the iteration count a guess and could overshoot the target.
