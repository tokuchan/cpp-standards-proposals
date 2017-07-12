Constraining Concepts Overload Sets
===================================

ISO/IEC JTC1 SC22 WG21 PXXX

    ADAM David Alan Martin  (adam@recursive.engineer)
    Nathan Myers            (ncm@cantrip.org)

// Check If Bjarne wants to join us (based upon dinner)

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
    // Assume a concept: "ConstIndexable"


    template< typename Key, typename Value >
    class ConstOnlyMap
    {
        private:
            // ...

        public:
            const Value &operator [] ( const Key &k ) const;
    };

    template< typename Key, typename Value >
    class MutableOnlyMap
    {
        private:
            // ...

        public:
            Value &operator [] ( const Key &k );
    };

    template< typename Key, typename Value >
    class WideMap
    {
        private:
            // ...

        public:
            const Value &operator [] ( const Key &k ) const;
            Value &operator [] ( const Key &k );
    };


    template< ConstIndexable CI, Key K >
    auto
    dangerous( CI &container, K key )
    {
        return container[ key ];
    }
    ~~~


The Issue
---------

The issue is in what kind of container we pass to `dangerous`.  If we pass a `ConstOnlyMap`, then no
unauthorized access to the container will happen.  If we pass `MutableOnlyMap`, it is unclear (to me
at this time -- have to find this out) whether the definition of the concept will pass or fail.  The
most dangerous case is `WideMap`.  `WideMap` unambiguously provides a `const`-member overload and
therefore satisfies the definition completely.  The problem is that when the code for
`dangerous< WideMap< Key, Value >, Key >` is generated, it will be generated as a normal template,
with no special consideration given to the constraints of the concept.

From the perspective of the compiler, the way it will actually compile the function `dangerous` is
as if it were written:

    ~~~
    template< typename CI, typename K >
    auto
    dangerous( CI &container, K key )
    {
        return container[ key ];
    }
    ~~~


At this point, the invocation of `CI::operator[]` will be whatever best matches `decltype( container )`,
which is the non-const overload.

When this happens, the container will actually be modified, which is contrary to the original intent
of the author of `ConstIndexable`.  If, in a future standard, we were decide to repair this oversight,
we are left with one of two incredibly unpallateable alternatives:

 1. Make constraint violations an error.
 2. Make the code do something different.

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

This makes it starkly apparent that Concepts must have overload constraint support from the onset, otherwise
the language will never get it -- the opportunity will be lost.


Other Unforseen Examples
------------------------
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
   did have this feature.

We propse that the Concepts feature is incomplete without constrained overload set for usage.  It is
vital that we explore this issue.
