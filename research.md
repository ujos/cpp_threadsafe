
# Finding thread race conditions at compile time

## Background

* There is an object. Object has state. State is deterministic and cannot be invariant after calling class methods.
* Object state can consist of two or more independent sub-states (for example, incoming and outgoing queues, which do not depend on each other).
  Real effective state in this case is a multiplication of sub-states.
* There are methods/functions, which read/modify state
* There are threads which call methods/functions at the same or unique time

Thus, relation is as following: Threads -> Functions -> Data

* Data can be shared (between two or more threads) or not shared
* Function can be threadsafe or not. For example:
  ```cpp
    class X {
        std::mutex m_;
        int i;

        // This function is not thread safe. After calling          
        // it in two threads value can be incremented by 1 (not 2).
        void foo() {        
                            
            ++i;
        }


        // This function is thread safe. After calling it in two
        // threads value will be incremented by 2 always.              
        void foo_threadsafe() {         
            m_.lock();
            foo();
            m_.unlock();
        }
    };
  ``` 

## Problems

1. User calls methods without insurance of thread safety. 
   Causes data races.
2. Interleaving synchronized and unsynchronized functions calls in callstack. 
   Causes unnecessary locks and performance degradations.
3. Unintentional variables mixing, which is read and modified asynchronously.
   Causes false sharing and performance degradation.

## Cases

### Case #1

1. Object has 1 state: S1
2. Inside thread #1 Method A accesses S1
3. Inside thread #2 Method B accesses S1
4. Access to S1 must be synchronized

### Case #2

1. Object has 3 sub-states S1, S2, S3
2. Inside thread #1 Method A accesses S1 and S2
3. Inside thread #2 Method B accesses S2 and S3
4. Access to S2 must be synchronized. S1 and S3 can be kept without synchronization.

### Case #3

1. Object has state, which consists of few thread safe objects. 
2. Access to those objects still must be synchronized to keep final state consistent.

### Case #4

1. Regardless of found solution, there are legacy classes with thread safe interfaces.
2. It is must be possible to use legacy classes in new environment as is

### Case #5

1. Methods inside class do not use locks. Instead they use lock free + wait free approaches. Thus, there are no "lock/unlock" methods.

### Case #6

1. Methods inside class do not use locks. Instead they use lock free. Thus, there are no "lock/unlock" methods.

## Solution
                       
New qualifier should be introduced - `shared` - which can be applied to:

* class data members
* variables
* parameters to methods/functions

If applied, it means that resource is shared between threads. Access to this resource must be synchronized.

New qualifier should be introduced - `threadsafe` - which can be applied to:

* methods - indicating that `this` points to`shared` object
* class data members - indicating that this field is self-synchronized and does not depend on other fields.

New qualifier should be introduced - `???` - which will be applied to fields/variables/parameters of legacy classes
If applied, all public methods of this class are considered as `threadsafe`.

### Usages

- If variable is `shared`, only `threadsafe` methods and fields are accessible.

  ```cpp
    struct X {
        Z f1;
        Z foo();

        threadsafe Z f2;
        Z bar() threadsafe;
    };

    shared X x;

    shared Z* ptr = &x.f1;              // ERROR
    std::cout << x.f1 << std::endl;     // ERROR
    std::cout << x.foo() << std::endl;  // ERROR
    std::cout << x.f2 << std::endl;     // OK
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
## Links

* ["volatile: The Multithreaded Programmer's Best Friend" By Andrei Alexandrescu (February 01, 2001)](http://www.drdobbs.com/cpp/volatile-the-multithreaded-programmers-b/184403766)