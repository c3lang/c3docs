## A quick primer on C3 for C programmers

This primer is intended as a guide to how the C syntax – and in some cases C semantics
– is different in C3. It is intended to help you take a piece of C code and understand
how it can be converted manually to C3.

#### Struct, enum and union declarations

Don't add a `;` after enum, struct and union declarations, and note the slightly
different syntax for declaring a named struct inside of a struct.

    // C
    typedf struct
    {
      int a;
      struct 
      {
        double x;
      } bar;
    } Foo;

    // C3
    struct Foo
    {
      int a;
      struct bar 
      {
        double x;
      }
    }

Also, user defined types are used without a `struct`, `union` or `enum` keyword, as 
if the name was a C typedef.

#### Arrays

Array sizes are written next to the type and arrays do not decay to pointers,
you need to do it manually:

    // C
    int x[2] = { 1, 2 }; 
    int *y = x;

    // C3
    int[2] x = { 1, 2 };
    int *y = &x;

You will probably prefer slices to pointers when passing data around:

    // C
    int x[100] = ...;
    int y[30] = ...;
    int z[15] = ...;
    sortMyArray(x, 100);
    sortMyArray(y, 30);
    // Sort part of the array!
    sortMyArray(z + 1, 10); 

    // C3
    int[100] x = ...;
    int[30] y = ...;
    sortMyArray(&x); // Implicit conversion from int[100]* -> int[] 
    sortMyArray(&y); // Implicit conversion from int[30]* -> int[]
    sortMyArray(z[1..10]; // Inclusive ranges!

Note that declaring an array of inferred size will look different in C3:

    // C
    int x[] = { 1, 2, 3 }; // x is int[3]

    // C3
    int[*] x = { 1, 2, 3 }; // x is int[3]

Arrays are trivially copyable:

    // C
    int x[3] = ...;
    int y[3];
    for (int i = 0; i < 3; i++) y[i] = x[i];

    // C3
    int[3] x = ...;
    int[3] y = x;

See more [here](../arrays).

#### Undefined behaviour

C3 has less undefined behaviour, in particular integers are defined as using 2s
complement and signed overflow is wrapping. See more [here](../undefinedbehaviour).

#### Functions

Functions are declared like C, but you need to put `fn` in front:
   
    // C:
    int foo(Foo *b, int x, void *z) { ... }

    // C3
    fn int foo(Foo* b, int x, void *z) { ... }

See more about functions, like named and default arguments [here](../functions).

#### Calling C functions

Declare a function (or variable) with `extern` and it will be possible to
access it from C3:

    // To access puts:
    extern fn int puts(char*);
    ...
    puts("Hello world");

Note that currently only the C standard library is automatically passed to the linker. 
In order to link with other libraries, you either need to explicitly tell 
the compiler to link them.

If you want to use a different identifier inside of your C3 code compared to
the function or variable's external name – use the `@extname` attribute:

    extern fn int _puts(char* message) @extname("puts");
    ...
    _puts("Hello world"); // <- calls the puts function in libc

#### Identifiers

Name standards are enforced:

    // Starting with uppercase and followed somewhere by at least
    // one lower case is a user defined type:
    Foo x;
    M____y y;
    
    // Starting with lowercase is a variable or a function or a member name:

    x.myval = 1;
    int z = 123;
    fn void fooBar(int x) { ... }

    // Only upper case is a constant or an enum value:

    const int FOOBAR = 123;
    enum Test 
    {
      STATE_A = 0,
      STATE_B = 2
    }    

#### Variable declaration

Declaring more than one variable at a time is not allowed:

    // C
    int a, b; // Not allowed in C3

    // C3
    int a;
    int b;

In C3, variables are always zero initialized, unless you explicitly opt out using `void`:

    // C
    int a = 0;
    int b;

    // C3
    int a;
    int b = void;

#### Casts

Cast syntax is slightly different:

    // C
    int a = (int)foo();

    // C3
    int a = (int)(foo());

You can also cast one struct to another as long as they are structurally equivalent:

    struct Foo { int a; int b; }
    struct Bar { int x; int y; }

    Foo f = { 1, 2 };
    Bar b = (Bar)(f);

#### Compound literals

Compound literals look different, but assigning to a struct will infer the type even if
it's not the initializer.

    // C
    Foo f = { 1, 2 };
    f = (Foo) { 1, 2 };
    callFoo((Foo) { 2, 3 });

    // C3
    Foo f = { 1, 2 };
    f = { 1, 2 };
    callFoo(Foo({ 2, 3 }));


#### Typedef

Instead of `typedef`, use `define`

    // C
    typedef Foo* FooPtr;

    // C3
    define FooPtr = Foo*;

`define` also allows you to do things that otherwise you'd use `#define` for:

    // C
    #define puts println
    #define my_string my_excellent_string 
    
    char *my_string = "Party on";
    ...
    println(my_excellent_string);

    // C3
    define println = puts;
    define my_excellent_string = my_string;
    
    char* my_string = "Party on";
    ...
    println(my_excellent_string);

Read more about `define` [here](../define).

#### Basic types

Several C types that would be variable sized are fixed size, and others changed names:

    // C
    int16_t a;
    int32_t b;
    int64_t c;
    uint64_t d;
    size_t e;
    ssize_t f;
    ptrdiff_t g;
    intptr_t h;

    // C3
    short a;    // Guaranteed 16 bits
    int b;      // Guaranteed 32 bits
    long c;     // Guaranteed 64 bits
    ulong d;    // Guaranteed 64 bits
    usize e;    // Same as C size_t, depends on target
    isize f;    // Same width as usize but signed
    iptrdiff g; // Same as C ptrdiff_t depends on target
    iptr h;     // Same as intptr_t depends on target
    ireg i;     // Register sized integer

Read more about types [here](../types).

#### Instead of #include: Modules and import

Modules are not mandatory but create a namespace, to import the names from a module, 
use `import`:

    module mylib::foo;
    
    fn void test() { ... }
    struct FooStruct { ... }

    module mylib::bar;
    import mylib::foo;
   
    fn void myCheck()
    {
      foo::test(); // foo prefix is mandatory.
      mylib::foo::test(); // This also works;
      FooStruct x; // But user defined types don't need the prefix.
      mylib::foo::FooStruct y; // But it is allowed.
    }


#### Comments

The `/* */` comments are nesting:

    /* This /* will all */ be commented out */

Note that doc comments, starting with `/**` has special rules for parsing it, and is
not considered a regular comment. See [preconditions](../contracts) for more information.

#### Type qualifiers

Qualifiers like `const` and `volatile` are removed, but `const` before a constant
will make it treated as a compile time constant. The constant does not need to be typed.

    const A = false;
    // Compile time 
    $if (A):
      // This will not be compiled
    $else:
      // This will be compiled
    $endif

`volatile` is replaced by macros for volatile load and store.

#### Goto removed

`goto` is removed, but there is labelled `break` and `continue` as well as `defer`
to handle the cases when it is commonly used in C.

    // C
    Foo *foo = malloc(sizeof(Foo));
    
    if (tryFoo(foo)) goto FAIL;
    if (modifyFoo(foo)) goto FAIL;

    free(foo);
    return true;

    FAIL:
    free(foo);
    return false; 

    // C3
    Foo *foo = @mem::malloc(Foo);
    defer free(foo);

    if (tryFoo(foo)) return false;
    if (modifyFoo(foo)) return false;

    return true;


#### Changes in `switch` 

`case` statements automatically break. Use `nextcase` to fallthrough to the 
next statement, but empty case statements have implicit fallthrough:

    // C
    switch (a)
    {
      case 1:
      case 2:
        doOne();
        break;
      case 3:
        i = 0;
      case 4:
        doFour();
        break;
      case 5:
        doFive();
      default:
        return false;
    }

    // C3
    switch (a)
    {
      case 1:
      case 2:
        doOne();
      case 3:
        i = 0;
        nextcase;
      case 4:
        doFour();
      case 5:
        doFive();
        nextcase;
      default:
        return false;
    }

Note that we can jump to an arbitrary case using C3:

    // C
    switch (a)
    {
      case 1:
        doOne();
        goto LABEL3;
      case 2:
        doTwo();
        break; 
      case 3:
    LABEL3:
        doThree();
      default:
        return false;
    }

    // C3
    switch (a)
    {
      case 1:
        doOne();
        nextcase 3;
      case 2:
        doTwo();
      case 3:
        doThree();
        nextcase;
      default:
        return false;
    }


#### Other changes

The following things are enhancements to C, that does not have a direct counterpart in
C. 

- [Expression blocks](../statements)
- Defer
- [Methods](../functions)
- [Errors](../errorhandling)
- [Semantic macros](../macros)
- [Generic modules](../generics)
- [Contracts](../preconditions)
- [Reflection](../reflection)
- Macro methods
