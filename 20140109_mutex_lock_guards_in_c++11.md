The new concurrency library of C++11 comes with two different classes for managing mutex locks: namely `std::lock_guard` and `std::unique_lock`. How do they compare with each other and in which situation which of the two classes should be preferably used?

The `std::lock_guard` class keeps its associated mutex locked during the entire life time by acquiring the lock on construction and releasing the lock on destruction. This makes it impossible to forget unlocking a critical section and it guarantees exception safety because any critical section is automatically unlocked when the stack is unwound after an exception was thrown. The `std::lock_guard` class should be used when a limited scope, like a class method, is to be locked.

``` cpp
void Foo::Bar()
{
    std::lock_guard<std::mutex> guard(this->Mutex);
    // mutex is locked now
}   // mutex is unlocked when lock guard goes out of scope
```

In contrast, the `std::unique_lock` class is a lot more flexible when dealing with mutex locks. It has the same interface as `std::lock_guard` but provides additional methods for explicitly locking and unlocking mutexes and deferring locking on construction. By passing `std::defer_lock` instead of `std::adopt_lock` the mutex remains unlocked when a `std::unique_lock` instance is constructed. The lock can then be obtained later by calling `lock()` on the `std::unique_lock` instance or alternatively, by passing it to the `std::lock()` function. To check if a `std::unique_lock` currently owns its associated mutex the `owns_lock()` method can be used. Hence, the mutex associated with a `std::unique_lock` doesn't have to be locked (sometimes also referred to as *owned*) during the lock guard's entire life time. As a consequence, the ownership of a `std::unqiue_lock` can be transferred between instances. This is why `std::unique_lock` is *movable* whereas `std::lock_guard` is not. Thus, more flexible locking schemes can be implemented by passing around locks between scopes.  
For example a `std::unique_lock` can be returned from a function, or instances of a class containing a `std::unique_lock` attribute can be stored in containers. Consider the following example in which a mutex is locked in the function `Foo()`, returned to the function `Bar()` and only then unlocked on destruction.

``` cpp
std::mutex Mutex;

std::unique_lock<std::mutex> Foo()
{
    std::unique_lock<std::mutex> lock(Mutex);
    return lock;
    // mutex isn't unlocked here!
}

void Bar()
{
    auto lock = Foo();
}   // mutex is unlocked when lock goes out of scope
```

Keeping `std::unique_lock`'s additional lock status up-to-date induces some additional, minimal space and speed overhead in comparison to `std::lock_guard`. Hence, as a general rule, `std::lock_guard` should be preferably used when the additional features of `std::unique_lock` are not needed.