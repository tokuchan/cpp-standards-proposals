


Conjuctive concepts further constrain the eligibility of a type, disjunctive concepts relax the
eligibility of a type.  This implies that overloads which are found in satisfying any branch
of a disjunction ought to be additive to the set of viable overloads.

When creating disjunctive constraints, it is important that the constrained function only call functions
whose names are in the intersection of the available names; however, we will not check this statically,
as that constitutes a form of definition checking which runs into the halting problem.  Any constrained
function which invokes calls to names in the intersection between the disjoint concepts must have
access to the union of the overloads available for that name.  The reason for this is that disjunction
can only reasonably be used to specify the optional existence of potential optimization opportunities
for any names in common between the two constraints.  This, from a design perspective of individual concepts,
further implies that the correct usage of disjunction for adjunct specializations should be formulated
as ( A && ( A || B ) ), not ( A || B ).  The truth table for ( A && ( A || B ) ) is the same as ( A ||
( A && B ) ) which is actually that of just "A".  This is because "A" was always required.


A B R
0 0 0
0 1 0
1 0 1
1 1 1

Conjuctive concepts further constrain the eligibility of a type when passed to a constrained function,
disjunctive concepts relax the eligibility of a type.  Because of this overloads which are found to
satisfy any branch of a disjunction ought be added to the set of viable overloads.  This is because
either side of the branch might be taken, or both, and thus whichever constraints are satisified should
have their functionality available.


