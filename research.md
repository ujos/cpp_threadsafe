
# Finding thread race conditions at compile time

## Background

* There is an object. Object has state. State is deterministic and cannot be invariant after calling class methods.
* Object state can consist of two or more independant sub-states (for example, incoming and outgoing queues, which do not depend on each other).
  Real effective state in this case is a multiplcation of sub-states.
* There are methods/functions, which read/modify state
* There are threads which call methods/functions at the same or unique time

Thus, relation is as following: Threads -> Instructions -> Data

* Data can be shared (between two or more threads) or not shared
* Instruction can be threadsafe or not

## Problems

1. User calls methods without insurance of thread safety. 
   Causes thread and data races.
2. Mixing synchronized and unsynchronized methods calls in callstack. 
   Causes unnessessary locks and performance degradations.
3. Unintentional data mixing, which is read and modified asynchronously.
   Causes false sharing and performance degradation.

## Cases

### Case #0

1. Object has 1 state: S1
2. Inside thread #1 Method A accesses S1
3. Inside thread #2 Method B accesses S1
4. Access to S1 must be synchronized


### Case #1

1. Object has 3 sub-states S1, S2, S3
2. Inside thread #1 Method A accesses S1 and S2
3. Inside thread #2 Method B accesses S2 and S3
4. Access to S3 must be synchronized. S1 and S2 can be kept withouot synchronization.

### Case #2

1. Object has state, which consists of few thread safe objects. 
2. Access to those objects still must be synchronized to keep final state consistent.

### Case #3

1. Regardless of found solution, there are legacy classes with thread safe interfaces.
2. It is must be possible to use legacy classes in new environment as is

### Case #4

1. Methods inside class do not use locks. Instead they use lock free + wait free approaches. Thus, there are no "lock/unlock" methods.

### Case #6 

1. Methods inside class do not use locks. Instead they use lock free. Thus, there are no "lock/unlock" methods.

## Solution
                       
New qualifier should be introduced - `shared` - which can be applied to:

* class fields
* variables
* parameters to methods/functions

If applied, it means that resource is shared between threads. Access to this resource must be synchronized.

New qualifier should be introduced - `threadsafe` - which can be applied to:

* methods - indicating that `this` parameter is `shared`
* class fields - indicating that this field is self-synchronized and does not depend on other fields.

New qualifier should be introduced - `???` - which will be applied to fields/variables/parameters of legacy classes
If applied, all public methods of this class are considered as `threadsafe`.

### Notes

- If variable is `shared`, only `threadsafe` methods and fields are accessible.

  ```cpp
    struct X {
        int i;
        int foo();

        threadsafe int j;
        int bar() threadsafe;
    };

    shared X x;

    shared int* ptr = &x.i;             // ERROR
    std::cout << x.i << std::endl;      // ERROR
    std::cout << x.foo() << std::endl;  // ERROR
    std::cout << x.j << std::endl;      // OK
    std::cout << x.bar() << std::endl;  // OK
  ```

  Another example: If method is `threadsafe`, `this` is `shared`. Thus, access to nested fields is forbidden.

  ```cpp
    struct Y {
        int foo() threadsafe;
    };

    struct X {
        Y y1;
        threadsafe Y y2;
        
        int bar1() threadsafe { 
            return y1.foo();             // ERROR
        }

        int bar2() threadsafe { 
            return y2.foo();             // OK
        }
    };
  ```

- Implicit pointer to `shared` object into pointer to non-`sharerd` object convertion is not allowed.
  ```cpp
    shared X x;
    X& x2 = x;                          // ERROR
    X* x3 = &x;                         // ERROR
  ```

- `const_cast`(???) should be used to convert `shared` into non-`shared`
  ```cpp
    shared X x;
    X& x2 = const_cast<shared X&>( x );     // OK
    X* x3 = const_cast<shared X*>( &x );    // OK
  ```

- It is not allowed to implicitly convert non-`shared` variable to `shared`.
  ```cpp
    X x;
    shared X& x2 = x;     // OK
    shared X* x3 = &x;    // OK
  ```

- It is allowed to combine `shared` with `threadsafe`. In this case 
  ```cpp
  class Queue {
  public:
    void pop() threadsafe;
    size_t count();
  };

  //

  struct X {
    shared threadsafe Queue queue_;  
  };

  X x1;
  x1.queue_.pop();       // OK. `queue_` is `shared` and `queue_.pop()` is `threadsafe`
  x1.queue_.count();     // ERROR: `queue_` is `shared`, however `count` is not `threadsafe`

  shared X x2;
  x2.queue_.pop();       // OK. 1) `queue_` is `threadsafe` (so we can access it). 2) `queue_` is explicitly `shared` and `queue_.pop()` is `threadsafe`
  x2.queue_.count();     // ERROR: 1) `queue_` is `threadsafe` (so we can access it). 2) `queue_` is `shared`, however `count` is not `threadsafe`

  //

  struct Y {
    threadsafe queue_;  
  };

  Y y1;
  y1.queue_.pop();       // OK
  y1.queue_.count();     // OK

  shared Y y2;
  y2.queue_.pop();       // OK. 1) `queue_` is `threadsafe` (so we can access it). 2) `queue_` is implicitly `shared` and `queue_.pop()` is `threadsafe`
  y1.queue_.count();     // ERROR: 1) `queue_` is `threadsafe` (so we can access it). 2) `queue_` is implicitly `shared`, however `count` is not `threadsafe`


  //

  struct Z {
    shared queue_;  
  };

  Z z1;
  z1.queue_.pop();       // OK. `queue_` is `shared` and `queue_.pop()` is `threadsafe`
  x1.queue_.count();     // ERROR: `queue_` is `shared`, however `count` is not `threadsafe`

  shared Z z2;
  z2.queue_.pop();       // FAIL. `queue_` is not explicitly `threadsafe` 
  z2.queue_.count();     // FAIL. `queue_` is not explicitly `threadsafe`

  ```
  