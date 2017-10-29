# Introduction
Many C\++ programmers are not aware of a big difference in how the *custom deleters* of `std::unique_ptr` and `std::shared_ptr` are implemented in the C\++ standard library. `std::unique_ptr` carries the custom deleter as part of its type (`template<class T, class Deleter> std::unique_ptr`). In contrast, the custom deleter of `std::shared_ptr` is not part of the type (`template<class T> std::shared_ptr`) but part of the constructor's template argument list (`template<class Y, class Deleter> shared_ptr(Y *ptr, Deleter d)`). Mostly, this difference doesn't matter much. Though, there are use-cases, like e.g. factories returning `std::unique_ptr` with a custom deleter, were the difference does matter.

# Design choices
## `std::unique_ptr`
The advantage of making the custom deleter part of `std::unqiue_ptr`'s type is that, as long as the deleter is stateless (e.g. a lambda that doesn't capture anything or a function with no member variables), storing it doesn't take up any additional memory thanks to the *empty base optimization*. This makes `std::unique_ptr` a *zero-overhead* abstraction, which means that:

1. Its size is identical to the size of a raw pointer on the underlying architecture.
2. All calls to the deleter can be inlined.

One possible implemention which makes use of the empty base optimization is to store the wrapped pointer together with the deleter in a [*compressed pair*](http://www.boost.org/doc/libs/1_61_0/libs/utility/doc/html/compressed_pair.html). The obvious disadvantage of making the custom deleter part of the type is that two `std::unique_ptr`s  with different custom deleters are of different type, even if they wrap the same pointer type.

## `std::shared_ptr`
In contrast to `std::unique_ptr`, `std::shared_ptr` provides the convinience of a *type erased* deleter. Type erased means that the type of the custom deleter is not dragged into `std::shared_ptr`'s type. Hence, one cannot know by just looking at the type if two `std::shared_ptr` instances have different custom deleters or not.
The type erasure makes `std::shared_ptr` more flexible. For example changing the allocation strategy of a factory, and with it the custom deleter of the returned `std::shared_ptr`s, doesn't break source/binary compatibility and thereby, doesn't require any recompilation of client software.
The drawback is that storing the custom deleter takes up additional memory, because some wrapper (e.g. `std::function` or a raw function pointer) is needed to store the custom deleter. The rationale behind this design choice is that `std::shared_ptr` must anyways heap allocate memory for its shared *control block*, containing the wrapped pointer and the reference counter. Additionally including the custom deleter didn't seem like a big cost, taking the increased flexiblity into account.

# Type erased custom deleters with `std::unique_ptr`
Imagine you're building an object factory which returns `std::unique_ptr`s. The return type of the factory's `Create()` function must allow casting instances of different derived classes to the same `std::unique_ptr` type. One way to do that is to use a `std::unique_ptr` to the base class. This, however, requires the base class' destructor to be `virtual`. What if the destructor cannot be `virtual` for some reason or the implications for source and binary compatibility are limiting?

An alternative is to create a type erased custom deleter for `std::unique_ptr` by wrapping the deleter e.g. in an `std::function<void(void *)>`. The wrapped function is then responsible for casting the `void *` to the correct type when deleting it. This construction works for virtual and non-virtual classes as well as for multiple inheritance, because the deleter casts the `void *` argument containing the address to the most derived class simply back to the type of the most derived class.

``` cpp
template<typename Type>
void MyDelete(void *ptr) // Casts 'ptr' to real type and deletes it
{
    delete static_cast<Type *>(ptr);
}

auto Factory::Create(int typeId)
{
    // Unique pointer with type erased custom deleter
    using UniquePtr = std::unique_ptr<Base, std::function<void(void *)>>;

    switch (typeId)
    {
    case 0: return UniquePtr(new Derived0, MyDelete<Derived0>); 
    case 1: return UniquePtr(new Derived1, MyDelete<Derived1>);
    // ...
   }
}
```

The applied type erasure doesn't come for free. There are two penalties to pay:

1. Destroying the pointer cannot be inlined anymore and therefore always requires an additional function call.
2. Additional memory is required to store the deleter.

It turns out that the `std::function` wrapper increases the memory footprint of the `std::unique_ptr` type considerably (32 bytes with GCC 5.3.1's libc\++ and 64 bytes with Visual C\++ 2015,  both 64 bit). Luckily, we can use a simple function pointer to reduce the total size of the final `std::unique_ptr` to 16 bytes.

```cpp
using UniquePtr = std::unique_ptr<Base, void(*)(void *)>;
```