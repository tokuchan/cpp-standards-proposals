Constraining Concepts Overload Sets
===================================

ISO/IEC JTC1 SC22 WG21 P0782R1

    ADAM David Alan Martin  (adam@recursive.engineer)
    Erich Keane             (erich.keane@intel.com)
	Hal Finkel
	Lisa Lippincott
	Gabriel Dos Reis

Abstract
--------

The concepts proposal does not constrain the overload set of a template-concept function.
This can lead to some surprising violations of the original concept author's expectations.
A constrained-overload set for use in name matching in a template is necessary to make this
safe; this was a part of the original C++0x concepts design and it appears to have been
forgotten.  Unfortunately, this is an oversight that cannot be corrected later -- to correct
this later would entail silent behavioral changes to far too much code that will exist using
concepts as we release them.  In other words, this is our only chance to get this right.

Simple Example
--------------

    ~~~
    // Assume a concept `Concept` which expects a member `f` taking one integer

    struct T1
    {
        void f( int );
    };

    struct T2
    {
        void f( double );
    };

    struct T3
    {
        void f( int );
        void f( double );
    };

    template< Concept C >
    void
    dangerous( C t )
    {
        t.f( 1.0 );
    }
    ~~~

The Issue
---------

The issue is in what kind of object we pass to `dangerous`.  If we pass a `T1`, then no surprises
happen -- the `double` is implicitly converted to `int`.  If we pass `T2`, the definition of the
concept will pass and an implicit conversion to `int` will not happen.  The most dangerous case
is `T3`.  `T3` provides a double and integer overload.  Although the concept requested an integer
like signature, the double signature will be called.

From the perspective of the compiler, the way it will actually compile the function `dangerous` is
as if it were written:

    ~~~
    template< typename CI >
    auto
    dangerous( CI &c )
    {
        return c.f( 1.0 )
    }
    ~~~


At this point, the invocation of `CI::f` will be whatever best matches `decltype( 1.0 )`,
which is the `double` overload.

If, in a future standard, we were to decide to repair this oversight, we are left with one of two incredibly
unpallateable alternatives and one unsatisfying one:

 1. Make constraint violations an error.
 2. Make the code do something different.
 3. Introduce a new syntax for indicating that a function definition is to be checked. (Defn' checking?)

In the first case, user code will break with obvious noises.  After that point, the user code would need
to be rewritten:

    ~~~
    template< ConstrainedKey CI, Key K >
    auto
    dangerous( CI &container, K key )
    {
        return std::as_const( container )[ std::as_const( key ) ];
    }
    ~~~

Although I am the author of `as_const`, this is not what I had in mind when I proposed it.  This change would
not be well received by users and would almost certainly be stillborn.

In the second case, some code will break noisily when suitable overloads do not exist, and further user code
will silently change behavior to better overloads when they exist.  Although this behavior is arguably more
correct from a purist point of view, the instability of behavior across two standards will also be stillborn.

The third alternative will not be dead-on-arrival, but it will mean that a new syntax for functions will
be necessary.  Such a syntax will have to be distinct from whatever "preferred" and "natural" syntax comes
with concepts in an earlier standard.

This makes it starkly apparent that Concepts must have overload constraint support from the onset, otherwise
the language will never get it -- the opportunity will be lost.


Other Unforseen Examples
------------------------
    ~~~
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
    ~~~

In this example, a the container will find itself adding multiple elements or with insane undefined behavior.
This is because the `Janus` element he expected to have converted would preferentially be turned into an
iterator instead of a `Member`.  The two-argument form of `insert` has a shorter conversion path, making it
a better match.  This yields a behavior which is drastically different than that expected by the user
in this example.

    ~~~
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
    ~~~


    There are also subtleties that will arise in addition to const when dealing with this problem:

    ~~~
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
    ~~~

There are other cases where this can cause incorrect lookup.  This fails to deliver upon a big part of the
expected benefits of a new language feature.  The comparison has been drawn between C++ Virtual Functions
and Concepts.  As concepts are being presented to bring generic programming to the main stream,
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
   concepts proposal lacks this and this oversight has yet to be discussed.  The original C++0x concepts
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

