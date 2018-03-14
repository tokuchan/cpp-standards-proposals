Constraining Concepts Overload Sets
===================================

ISO/IEC JTC1 SC22 WG21 P0782R1

    ADAM David Alan Martin  (adam@recursive.engineer)
    Erich Keane             (erich.keane@intel.com)

Abstract
--------

The central purpose of Concepts is to simplify generic programming such that it is approachable to the
non-expert developer.  In general it makes great strides towards this end particularly in the capacity
of invoking a generic function; however, the Concepts design does not deliver on this promise in the
implementation of a generic function.  This is because the feature does not constrain the overload set
of a template-concept function itself.  This is contrary to the expectations of non-experts, because
to them Concepts should strongly resemble the callable properties of an interface.  This mental model
drives their expectations to believe that Concepts offer a mechanism to limit the set of operations
which would be visible from within their constrained function to those which are specified by concept
used by the constrained function.

The fact that this is not the case in constrained functions can lead to surprising violations of
the author's expectations thereof.  Unfortunately, this oversight cannot be corrected later.  To correct
this later would entail silent behavioral changes to existing code after the release of Concepts in a
standard.  In other words, this is our only chance to get this right.


Simple Motivating Example
-------------------------

```
// Assume a concept, `IntegerSerializer` which requires a member function called `serialize` which
// takes an `int` and returns a `std::string` with the representation of that integer, in the format
// appropriate to the implementation.

class SimpleSerializer {
    std::string serialize( int value );
};

struct PrecisionSerializer {
    std::string serialize( double preciseValue );
};

struct LoanInterestSerializer {
    std::string serialize( int interestBasisPoints );
    std::string serialize( double interestRate );
};

std::string formatLogarithmicValue( IntegerSerializer &serializer, int integerValue ) {
    return serializer.serialize( std::log( integerValue ) );
}
```

The issue is in what kind of object we pass to `formatLogarithmicValue`.  If we pass an `SimpleSerializer`,
then no surprises happen -- the `double` is implicitly converted to `int`.  If we pass `PrecisionSerializer`,
then the definition of the concept will pass and an implicit conversion to `int` will happen, which is
potentially surprising; however, many programmers are reasonably comfortable with the idea of fundamental
type conversions.  The most surprising case is that of `LoanInterestSerializer`.  `LoanInterestSerializer`
provides a `double` and an `int` overload.  Although the concept requested function with a signature that
accepts an `int`, the overload which accepts `double` in its signature will be called.

From the perspective of the compiler, the way it will actually compile the function `formatLogarithmicValue`
is as if it were written:

```
template< typename IntegerSerializer >
std::string
formatLogarithmicValue( IntegerSerializer &serializer, int integerValue ) {
    return serializer.serialize( std::log( integerValue ) );
}
```

At this point, the invocation of `IntegerSerializer::serializer` will be whatever best matches
`decltype( std::log( integerValue ) )`, which is the overload with `double` as its parameter.
This is likely surprising behavior to the author of `formatLogarithmicValue`, as well as the caller
of `formatLogarithmicValue`.  Both of these authors would expect that the constraints described by
the concept would be obeyed, yet paradoxically the overload which was not the best match for the
constraint was actually the overload that was actually invoked in the body of the "constrained"
function!

The only way for an author of such a constrained function to avoid this, at present, is to rewrite
`formatLogarithmicValue` in such a way as to prevent the incorrect lookup.  Unfortunately, this requires
a level of C++  expertise regarding name lookup and overload resolution which is at odds with the level
of expertise expected of the audience of Concepts, viz. the non-expert programmer.  Such a rewrite
might appear thus:

```
std::string
formatLogarithmicValue( IntegerSerializer &serializer, int integerValue ) {
    return std::as_const( serializer ).serialize( static_cast< int >(
            std::log( std::as_const( integerValue ) ) ) );
}
```

This does not appear to be code that would be expected of the audience targetted by Concepts.  Additionally,
although one of the authors of this paper is author of `std::as_const`, this is not the purpose nor audience
he had in mind when he proposed it.

An Example at Scale
-------------------

```
namespace ConceptLibrary {
    // Assume a concept, `Stringable` which requires a member function called `toString` which
    // when called returns a string representation of the object.
}

namespace PayrollLibrary {
	class Employee {
		private:
			std::string name;

		public:
			Employee( std::string initialName );

            // This satisfies the Stringable concept, and the author of this type knows that.
			std::string toString() const;

			bool operator == ( const Employee &rhs ) const;

            // The following functions are friend, to indicate that ADL is intended.

            // Terminate the specified employee's employment
            friend void fire( const Employee &emp );

            // Initiate the specified employee's employment
            friend void hire( Employee emp );

            // Returns true if the specified employee is employed and false otherwise.
            friend bool worksHere( const Employee &emp );
	};
}

namespace AlgorithmLibrary {
	// This "fires" off a stringable object to be processed.
	void fire( const ConceptLibrary::Stringable &s ) {
        std::cout << "I am interested in " << s.toString() << std::endl;
    }

	void
	printAll( const std::vector< ConceptLibrary::Stringable > &v ) {
		for( auto &&s: v ) fire( s );
	}
}

namespace UserProgram {
	void code() {
        std::vector< PayrollLibrary::Employee > team;
		team.emplace_back( "John Doe" );
		AlgorithmLibrary::printAll( team );
	}
}
```

+++ Reword
In this example the intent of the three separate authors is apparent.  The author of the `PayrollLibrary`
simply wished to afford his users the ability to represent employees at a company.  The author of
the `AlgorithmLibrary` simply wished to afford his users the ability to print printable things and
needed to write an internal helper method to better organize his code.  The author of `UserProgram`
naievely wished to combine these reusable components for a simple task.  However, a subtle behavior
of name lookup in function templates resulted in the termination of employees, rather than the intended
call to an unimportant implementation detail.  Recall that the author of `AlgorithmLibrary` is perhaps
unaware of the fact that types may exist which are `Stringable` yet interfere with the name he chose for
his internal implementation detail.

+++ Reword:
The authors of this paper recognize the importance of respecting the original intent of the programmers
of these components without burying them in the details of defensive template writing.  These kinds of
examples will come up frequently and perniciously in codebases which import third party libraries and
work in multiple groups each with different naming conventions.  Even without variance in naming conventions,
names that have multiple meanings to multiple people are likely to be used across disparate parts of
a codebase, and thus they are more likely to exhibit this pathological behavior.


Addressing the Problem
----------------------


Should the terse syntax be accepted into the current standard, without addressing this issue, then
future attempts to repair this oversight in the language specification, leave us with one of two
incredibly unpallateable alternatives and one unsatisfying one:

 1. Make constraint violations an error, thus requiring extreme verbosity.
 2. Silently change the meaning of existing code, with ODR implications, because these constrained functions
    are templates.
 3. Introduce a new syntax for indicating that a function definition should be processed in a manner which
    is more consistent with average programmer expectations; 

In the first case, vast amounts of code will fail to compile in noisy ways.  After that point, the user code
would need to be rewritten, in a manner similar to the necessary rewrites as described above. 

In the second case, massive fallout from ODR, silent subtle semantic changes, and other unforseen dangers
lie in wait.

In the third case, the benefits of the "natural" syntax are lost, as the best syntax for beginners is no
longer the natural syntax!  This obviously defeats the intended purpose of Concepts with a natural syntax.

Solution
--------


In the second case, some code will break noisily when suitable overloads do not exist, and further user code
will silently change behavior to better overloads when they exist.  Although this behavior is arguably more
correct from a purist point of view, the instability of behavior across two standards will also be stillborn.

The third alternative will not be dead-on-arrival, but it will mean that a new syntax for functions will
be necessary.  Such a syntax will have to be distinct from whatever "preferred" and "natural" syntax comes
with Concepts in an earlier standard.

This makes it starkly apparent that Concepts must have overload constraint support from the onset, otherwise
the language will never get it -- the opportunity will be lost.


Other Unforseen Examples
------------------------
    ```
    // Assume a concept: "Insertable", which requires a member function `insert` which takes
    // an iterator and an object for insertion.

    template< Insertable I, Iterator Iter, typename E >
    void
    addElement( I &x, Iter i, typename e )
    {
        x.insert( i, e );
    }

    // Assume a container that looks like `std::map` in terms of a two argument
    // `insert` overload.  One taking `Iterator` and `Element`, the other taking two
    // `Iterator`s
    template< typename Container >
    class Container
    {
        public:
            class iterator; // ...

        public:
            void insert( iterator pos, Element e );

    };

    // Assume a type which is convertible to an `Iterator` and also to an `Element`.
    class Janus
    {
        public:
            operator Container< Janus >::iterator (); // ...

            operator int (); // ...

            // ...
    };

    class Member
    {
        public:
            Member( int ); // ...
    };

    void
    danger()
    {
        Container< Member > c;
        // ...
        Janus j= /* ... */;
        addElement( c, c.begin(), j );
    }
    ```

In this example, a the container will find itself adding multiple elements or with insane undefined behavior.
This is because the `Janus` element he expected to have converted would preferentially be turned into an
iterator instead of a `Member`.  The two-argument form of `insert` has a shorter conversion path, making it
a better match.  This yields a behavior which is drastically different than that expected by the user
in this example.

    ```
    // Assume a concept: "Drawable", in this namespace
    // Assume a concept: "Rotatable", in this namespace
    // Assume a concept: "RotatableDrawable", in this namespace

    class Cowboy
    {
        public:
            enum GunState { in= 0, out= 1 };

        private:
            GunState gunState;

        public:
            inline friend void draw( Cowboy &c ) { c.gunState= GunState::out; }
            inline friend GunState draw( const Cowboy &c ) { return c.gunState; }

            void rotate() {}
    };

    class Shape
    {
        public:
            // ...

        public:
            inline friend bool draw( const Shape &s ) { /* graphics package details */ }

            void rotate() {}
    };

    template< RotatableDrawable RD >
    bool
    turnAndDraw( RD thing, bool redraw )
    {
        thing.rotate();
        if( redraw ) return draw( thing );
        return false;
    }
    ```


    There are also subtleties that will arise in addition to const when dealing with this problem:

    ```
    // Assume a concept: "Composable", in this namespace

    inline
    std::string
    compose( std::string l, std::string r );

    class Function
    {
        private:
            // ...

        public:
            inline friend Function compose( Function, Function );
    };

    class Unicode
    {
        public:
            // ...

        public:
            operator const std::string &() const;
    }; 

    template< Composable C >
    bool
    mix( C a, C b )
    {
        compose( a, b );
    }
    ```

There are other cases where this can cause incorrect lookup.  This fails to deliver upon a big part of the
expected benefits of a new language feature.  The comparison has been drawn between C++ Virtual Functions
and Concepts.  As Concepts are being presented to bring generic programming to the main stream,
it is vital that it have 3 core safety components (compared to similar features in Object Oriented code):

1. An object passed to a function meets the qualifications that a concept describes.  This is
   analagous to how a function taking a pointer or reference to a class has the parameter checked for
   substitutiablility.  The current Concepts proposal provides this.

2. An object passed to a function had its definition checked for correctness by the compiler.  This
   is analagous to how a class is checked for abstractness.  The current Concepts proposal lacks this,
   but some have claimed that it is trivial to add this.  Whether this is the case is beyond the scope
   of this paper.

3. An object passed to a function only has functions called on it that are described by its Concept.
   This is analagous to how a function taking a pointer to base is only allowed to call members of the
   base -- new interfaces added in the derived class are not considered for better matching.  The current
   Concepts proposal lacks this and this oversight has yet to be discussed.  The original C++0x Concepts
   did have this capability, but via a different mechanism.

We propse that the Concepts feature is incomplete without constrained overload set for usage.  It is
vital that we explore this issue.


What This Paper is  _<b><u>NOT</b></u>_  Proposing
--------------------------------------------------

 1. Full Static Definition Checking
 2. C++0x Concept Checking
 3. C++0x Concept Maps
 4. The generation of "invisible" proxy types
 5. The generation of "invisible" inline proxy functions
 6. Dynamic dispatch
 7. Function call tables
 8. Implicit generation of adaptors 
 9. Any form of extra code generation
10. Any form of extra type generation
11. Relying upon the optimizer to make any aspect of the code generated by this solution more efficient

Our Proposed Solution
---------------------

We propose a moderate alteration of the overload resolution rules and name lookup rules that
statically filters out some overloads based upon whether those functions are used to satisfy the
concept's requirements.  This constrained overload set hides some functions from visibility in
the definition of constrained functions.  No change should be necessary to the rules of name
lookup itself; however, our new overload resolution rules will affect the results of names
found by unqualified name lookup in constrained functions-- this is by design.


Specifically, our design is to change overload resolution to be the following (taken from cppreference.com's
description of the overload resolution rules):

<table border=10>
<tr>
<td>
##Current overload resolution
<td>
##Desired overload resolution
<tr>
<td valign=top>
<p>
Given the set of candidate functions, constructed as described above, the next step of overload
resolution is examining arguments and parameters to reduce the set to the set of viable functions
<p>
To be included in the set of viable functions, the candidate function must satisfy the following:
<p>
<ol>
<li> If there are M arguments, the candidate function that has exactly M parameters is viable
<li> If the candidate function has less than M parameters, but has an ellipsis parameter, it is viable.
<li> If the candidate function has more than M parameters and the M+1'st parameter and all parameters
that follow must have default arguments, it is viable. For the rest of overload resolution, the
parameter list is truncated at M.

<li> If the function has an associated constraint, it must be satisfied
	(since C++20)

<li> For every argument there must be at least one implicit conversion sequence that converts it to the
corresponding parameter.
<li> If any parameter has reference type, reference binding is accounted for at this step: if an rvalue
argument corresponds to non-const lvalue reference parameter or an lvalue argument corresponds to rvalue
reference parameter, the function is not viable.
<td valign=top width=50%>
<p>
Given the set of candidate functions, constructed as described above, the next step of overload
resolution is examining arguments and parameters to reduce the set to the set of viable functions

<p>
To be included in the set of viable functions, the candidate function must satisfy the following:
<p>
<ol>
<li> If there are M arguments, the candidate function that has exactly M parameters is viable
<li> If the candidate function has less than M parameters, but has an ellipsis parameter, it is viable.
<li> If the candidate function has more than M parameters and the M+1'st parameter and all parameters
that follow must have default arguments, it is viable. For the rest of overload resolution, the
parameter list is truncated at M.

<li> If the function has an associated constraint, it must be satisfied
	(since C++20)
<p>
<font color=green>
<li face="X">  If the call to the function is within a constrained template function and at least one of the
   parameters is a concept and the function was found by unqualified name lookup and the lookup found
   a name that is a member of or in the namespace of the concept, then the function must be one of
   the functions that was used in the satisfaction of the constraint of the concept parameter involved
   in the unqualified name lookup.
    (this paper)
   
</font>

<li> For every argument there must be at least one implicit conversion sequence that converts it to the
corresponding parameter.
<li> If any parameter has reference type, reference binding is accounted for at this step: if an rvalue
argument corresponds to non-const lvalue reference parameter or an lvalue argument corresponds to rvalue
reference parameter, the function is not viable.

</table>


Revision History
----------------

P0782R0 - A Case for Simplifying/Improving Natural Syntax Concepts (Erich Keane, A.D.A. Martin, Allan Deutsch)

The original draft explained the general problem with a simple example.  The motivation was to help attain
a better consensus for "Natural Syntax" Concepts.  It was presented in the Monday Evening session of EWG at
Jacksonville, on 2018-03-12.  The guidance from the group was strongly positive:
<p>
SF: 10 - F: 21 - N: 22 - A: 7 - SA: 1

