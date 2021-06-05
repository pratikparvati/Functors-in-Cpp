# Function Object {aka Functors} in C++

**Functors** in C++ are short for **function objects**. Function objects are instances of C++ classes that have the `operator()` defined, which gives the class object function semantics. In the class-like functions constructed in C++, the function is composed of the return type, function name, parameters, and function body. The basic accepted form of functors looks like this...

```C++ 
class Functor
{
public:
	R operator()(P1, ..., Pn)
	{
		return R();
	}
};
```
Where R is the return type, P is a parameter type and just like any function the number of parameters is arbitrary. The above code works because of the flexibility `operator()` provides in C++ which can take any number of arguments of any types and return anything it wants to (all the other operators have a fixed number of arguments).

### Motivation behind Functors

With the procedural programming, the big problems are broken down into small chuncks called functions, procedures or subroutines. This was generally called top-down or bottom-up modular programming and it was the best we had. The Object-oriented programming takes modular code and places it with the data that it works with, that is, objects have properties, also known as data (member variables), and methods that operate on that data, also known as methods (member functions).

A language that allows functions (something not connected to an object) to stand on their own can't really be called 100% object oriented. With the "everything is an object" approach in C++ we can achieve the functionality of standalone functions by the simple trick of allowing objects which are functions. 

#### Function pointer limitation in C

If you want to provide a callback mechanism in C, you must first implement a callback function and then pass the address of that function to the invoker (the code that will call your callback when it is needed). Unfortunately, there are several disadvantages to using C-style function pointers

1. A function has no instance state because there is no such thing as multiple instances of a function; there will always be one instance, and it is global. You could add `static` variables to your function to retain state, but since a function only has one instance there can only be once instance of the static member and that must be shared between function calls. (I will cover how functors solve this in later section)

```C++
MyClass& foo()
{
	static MyClass Obj;
	// some operation on Obj
	return Obj;
}
```
2. Function pointers don't play well with templates; if only one signature of the function exists, things will work; otherwise, the compiler will complain about template instantiation ambiguities, which can only be resolved by nasty function pointer casting.

```C++
void callback_func(void *){}
void callback_func(double *){}

template <typename T>
void invoker(T cb_func)
{
	// type checking and other stuffs
	cb_func();
}

int main()
{
	invoker(callback_func); // ERROR: Which callback_func?

	// nasty function pointer casting
	invoker((void (*)(void*))callback_func);  
	invoker((void (*)(double*))callback_func);
}
```
3.The compiler can inline calls to the functor; it cannot do the same for a function pointer. This is why C++ `std::sort()` beats C `qsort()` performance-wise.

### How to use functors?

Let's look at an example. This example creates a functor class with a constructor that takes an integer argument and saves it. When objects of the class are "called", it will return the result of mutiplying the saved value and the argument to the functor.

```C++
#include <iostream>

class MultFunctor
{
public:
    explicit MultFunctor(int x) : m_x(x) {}
    int operator()(int y)
    {
        return m_x * y;
    }

private:
    int m_x;
};

int main()
{
    MultFunctor multTen(10);
    std::cout << multTen(6);

    return 0;
}
```
Notice that even though `multTen` looks like a function it really is an object. That is `MultFunctor` can have properties, methods, constructors and destructors in addition to the function it implements and it can be passed to other functions just like any other object.

With the multiple call to `multTen()`, the data `m_x` remains the same i.e, the state of the function object is retained until the object is destroyed. We can also have multiple `MultFunctor` objects with its state retained until that object is destroyed; this gives a biggest advantage over function pointers.

Lets look into another example, consider the example of a sorting routine that uses a callback function to define an ordering relation between a pair of items. The following C program uses function pointers:

```C
#include <stdlib.h>

int integerCompare(const void* a, const void* b)
{
    return (*(int *)a - *(int *)b));
}

int main(void)
{
    int items[] = { 4, 3, 1, 2, 6, 8, 9, 7 };

    /**
	 * prototype of qsort is
	 * void qsort(void *base, size_t nel, size_t width, int (*compar)(const void *, const void *));
	 */
    qsort(items, sizeof(items) / sizeof(items[0]), sizeof(items[0]), integerCompare);
    return 0;
}
```
In C++ functors can be used instead of ordinary function as

```C++
class IntegerCompare
{
  bool operator()(const int &a, const int &b) const
  {
    return a < b;
  }
};

int main()
{
    std::vector<int> items { 4, 3, 1, 2, 6, 8, 9, 7 };
    std::sort(items.begin(), items.end(), IntegerCompare());
    return 0;
}
```
### Using Functors and Function pointers with templates

You cannot pass a functor as a function pointer into a function that takes a function pointer, even if the functor has the same arguments and return value as the function pointer. Similarly, if a function expects a functor, you cannot pass in a function pointer.

You must use templates if you want to allow either a function pointer or a functor to be passed into the same function. The templated function will determine the appropriate type for the functor or function pointer, and because both functors and function pointers are used in the same way—they (both look like function calls).

Below is example code to print the elements of vector using a template function that accepts both function pointer as well as function object as an argument.

```C++
#include <iostream>
#include <vector>
#include <iterator>
#include <algorithm>

typedef void (*printV)(std::vector<int> vec);

// callback function for print using function pointer
void printVector(std::vector<int> vec)
{
	/**
	 * The STL algorithm for_each accepts as parameters a range of iterators and a unary function,
     * then calls the function on each argument. 
	 */
    std::for_each(vec.begin(), vec.end(), [](int i)
                  { std::cout << i << ' '; });
}

// Functor class to print vector elements
class printVecFunctor
{
public:
    void operator()(std::vector<int> vec)
    {
        std::for_each(vec.begin(), vec.end(), [](int i)
                      { std::cout << i << ' '; });
    }
};

// template function that accepts both function pointers and functors
template <typename Tfunc>
void printVectorElements(Tfunc print, std::vector<int> vec)
{
    print(vec);
}

int main()
{
    std::vector<int> vec = {10, 20, 30, 40, 50};
    printV pV = printVector;
    std::cout << "Print using function pointer: ";
    printVectorElements(pV, vec);
    std::cout << '\n';

    printVecFunctor p{};
    std::cout << "Print using function object: ";
    printVectorElements(p, vec);

    return 0;
}
```
Functors are a strange but incredibly useful feature of the C++ language that are essentially “smart functions.” While initially functors can be a bit confusing, with practice you will come to appreciate their immense power, flexibility, and versatility. Knowing how to write and use functors is a key success factor in writing generic and reusable code and being able to make use of advanced features of the STL.



