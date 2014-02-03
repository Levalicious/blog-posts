Welcome to this quick post on constructor initialization lists in C++. If you are not familiar with initialization lists here is a very short introduction. When in C++ a class is instantiated, first, all base classes and second, all class attributes are constructed. Initialization lists are used to control that process. In initialization lists the constructors of the base class and the class attributes can be explicitly called. Otherwise, they are initialized by calling their default constructor. For efficiency reasons it is important to use initialization lists, because all member initialization takes place before the body of the constructor is entered. So much about the basics of initialization lists. Now a little riddle. The following piece of code will crash. Can you find the problem? Give it a try your-self before you continue reading.

``` cpp
struct Foo
{
    Foo(int newId) : id(newId) {}
    int id;
};

struct Bar
{
    Bar(const Foo &newFoo) : foo(newFoo), fooId(foo.id) {}

    const int   fooId;
    const Foo & foo;
};

int main(int argc, char **argv)
{
    Foo foo(303);
    Bar bar(foo);
    return 0;
}
```

If you don't know the nasty details of how class attributes are initialized via initialization lists, you most probably couldn't figure out what causes the crash in those few lines of code. The issue with the code above is that the order of attribute initialization is not determined by the order of appearance in the initialization list, but by the order of declaration within the class. Note, that `foo` appears in the initialization list before `fooId`, but `fooId` is declared before `foo`. Hence, not `foo` but `fooId` is initialized first, accessing the still uninitialized attribute `foo`.

So, why is that you might ask? This, on the first glance, strange behavior actually makes a lot of sense. When an object is destroyed, its destructor calls the destructors of every class attribute in the reverse order they were initialized. As there can be potentially more than one constructor with different orders of attribute initialization the order of destruction wouldn't be defined. To solve the ambiguity simply the order of declaration is used.