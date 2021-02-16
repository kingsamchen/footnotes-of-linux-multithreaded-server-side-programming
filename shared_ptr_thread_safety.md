# shared_ptr Thread Safety

`shared_ptr` objects offer the same level of thread safety as built-in types. A `shared_ptr` instance can be "read" (accessed using only const operations) simultaneously by multiple threads. Different `shared_ptr` instances can be "written to" (accessed using mutable operations such as `operator=` or `reset`) simultaneously by multiple threads (even when these instances are copies, and share the same reference count underneath.)

Any other simultaneous accesses result in undefined behavior.

Examples:

```c++
shared_ptr<int> p(new int(42));
```

Code Example 4. Reading a `shared_ptr` from two threads

```c++
// thread A
shared_ptr<int> p2(p); // reads p

// thread B
shared_ptr<int> p3(p); // OK, multiple reads are safe
```

Code Example 5. Writing different `shared_ptr` instances from two threads

```c++
// thread A
p.reset(new int(1912)); // writes p

// thread B
p2.reset(); // OK, writes p2
```

Code Example 6. Reading and writing a `shared_ptr` from two threads

```c++
// thread A
p = p3; // reads p3, writes p

// thread B
p3.reset(); // writes p3; undefined, simultaneous read/write
```

Code Example 7. Reading and destroying a `shared_ptr` from two threads

```c++
// thread A
p3 = p2; // reads p2, writes p3

// thread B
// p2 goes out of scope: undefined, the destructor is considered a "write access"
```

Code Example 8. Writing a `shared_ptr` from two threads

```c++
// thread A
p3.reset(new int(1));

// thread B
p3.reset(new int(2)); // undefined, multiple writes
```

Starting with Boost release 1.33.0, `shared_ptr` uses a lock-free implementation on most common platforms.

If your program is single-threaded and does not link to any libraries that might have used `shared_ptr` in its default configuration, you can #`define` the macro `BOOST_SP_DISABLE_THREADS` on a project-wide basis to switch to ordinary non-atomic reference count updates.

(Defining `BOOST_SP_DISABLE_THREADS` in some, but not all, translation units is technically a violation of the One Definition Rule and undefined behavior. Nevertheless, the implementation attempts to do its best to accommodate the request to use non-atomic updates in those translation units. No guarantees, though.)

You can define the macro `BOOST_SP_USE_PTHREADS` to turn off the lock-free platform-specific implementation and fall back to the generic `pthread_mutex_t`-based code.

---

1. 见书 P17
2. 原文链接：https://www.boost.org/doc/libs/release/libs/smart_ptr/doc/html/smart_ptr.html#shared_ptr_thread_safety