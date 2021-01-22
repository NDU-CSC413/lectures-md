---
title: C++ Review 
---

# Variables and references

When we define (declare) a variable the system reserves space in memory to store the
value associated with that variable. This is the reason why we need to specify the 
type of the variable since the required space depends on it. For example (typically), 
an ```int``` and ```float``` need 4 bytes whereas ```long``` and ```double``` need 
8 bytes. Because every variable is associated with a location in memory we can
determine the memory address where the variable is located using the & operator. Note 
that the & operator can have different meaning depending on context.
```cpp
#include <iostream>
int main(){
//a location in memory is reserved and labeled x
int x=2;
//y is just another name for the same location. no reservation is done.
int& y=x;
//a location is reserved for z and the value of x is copied
int z=x;
z=17;
y=13;
//print the value of the variables and their respective addresses
std::cout<<"x= "<<x<<" and <<"&x="<<&x<<std::endl;
std::cout<<"x= "<<y<<" and <<"&x="<<&y<<std::endl;
std::cout<<"x= "<<z<<" and <<"&x="<<&z<<std::endl;

}
```
Note that ```int& y=x;``` declares _y_ as a reference to _x_ whereas ```&x``` gives the memory address of _x_. The different declarations used above carry to the parameters in function calls. For example,

```cpp
#include <iostream>
void byValue(int n){
    n=17;
}
void byRef(int& n){
    n=12;
}
int main(){
  int x=2;
  byValue(x);
  std::cout<<x<<std::endl;
  byRef(x);
    std::cout<<x<<std::endl;

}
```

So in the call to the function ```byValue(x)``` it is __as if__ we declare ```int n=x;``` and therefore _n_ is a __copy__ of _x_. By contrast, ```byRef(x)``` is is __as if__ we declare ```int& n=x;``` so no copy is made and _n_ is a reference to _x_.
Usually we call by reference when either we want to change the input or  when the input is large and copying becomes expensive. We can use the best of both by using a const reference

```cpp
int byCRef(const int& n){
    n=37;//error cannot modify n
    return 2*n;
}
```
Also, const allows us to pass literals and temporaries.

```cpp
int byT(int n){
    return 7*n;
}
int byCRef(const int & n){
    n=n+1;//error n is const
    return 2*n;
}
int byRef(int& n){
    n=n+1;//changes the value of parameter
    return 2*n;
}
int main(){
    byRef(2);//error cannot bind a non-const lvalue to rvalue
    byCRef(2);//OK
    byRef(byT(2));//error since the return value of byT is a temp
    byCRef(byT(2));//OK
    int& r=byT(2);//error cannot bind 
    int&& res=byT(2);//ok
}
```

You can run the above code [here](https://godbolt.org/z/e8v8cx).

# Classes

In C++ new types are created using classes. Once a class is defined new objects can be instantiated from such a class. Minimal syntax of a (useless) class

```cpp
class Test{};
int main(){
    Test t;
}
```

A class can have __member variables__ and __member functions__.

```cpp
class Test{
    int _x;
    public:
    int& x(){
        return _x;
    }
    
};
int main(){
    Test t;// at this point _x is undefined
    t.x()=17;
}
```
By default all members of a class are __private__ and hence inaccessible from outside the scope of the object. To make a member accessible we use the keyword __public__. Note that the __member function__ ```x()``` returns a __reference__ to _x and this allows us to change the value of _x. We can use pointers as usual where the code below is equivalent to the one above but using the arrow instead of the dot operator.

```cpp
int main(){
    Test * p=new Test();
    p->x()=17;
}
```

In fact the private and public qualifiers can be used for any and all members. For example

```cpp
class Test {
    private: int _x;//_x is private
    public: int _y;// _y is public
     int _z;// _z is public. The keyword carries over until it changes
     private: void f(){}
     public: int& x(){return _x;}
}
```
But usually all public members are grouped together using a single keyword and the same for private members;

```cpp
int main(){
    Test t;
    t._x;//error _x is inaccessible
    t._y=t._z;//OK both are public
    t.x();//OK
    t.f();//Error f is inaccessible
}
```

## Constructors and destructors

For builtin types like int and double a variable is "created" (memory is reserved) when the variable is declared. Once the variable is out of scope is it "destroyed" (memory is released. The same thing is done for objects instantiated from classes. This is done by using __constructor__ and __destructor__. When one don't supply our own versions a default version is used by the compiler which basically calls the constructors and destructors of the member variables.

```cpp
class Test {
    public:
    int _x;
    double _y;
}
int main(){
    Test t;
    std::cout<<t._x<<std::endl;
    std::cout<<t._y<<std::endl;


}
```
No constructor is supplied so the compiler uses a default that creates variables _x and _y. The output is 
```cpp
0
0
```

# Rvalue reference
Since C++11 there is a new type of references called **rvalue** references.
The variable _res_ above extends the lifetime of the temporary object created by the ByT() function.  To see that consider when the destructor is called in the following code

```cpp
#include <iostream>
struct Test {
  int _x;
  Test(int x=0):_x(x){}
  ~Test(){
    std::cout<<"dtor "<<_x<<std::endl;
  }
};
Test RT(int val){
   return Test(val);
}
int main() {
Test&& res=RT(8);
std::cout<<"creating 7\n";
RT(7);
std::cout<<" done\n";

}
```
You can run the above code [here](https://godbolt.org/z/4EacM3)


Note that  when when an rvalue reference  is used, it is used as a lvalue reference. This is called **move semantics is not passed through** .For the example the following recursive function gives an error

```cpp
void doit(std::string&& s){
  if(s!="hello")
    doit(s);

}
```
this is a fix

```cpp
void doit(std::string&& s){
  if(s!="hello"){
    s="hello"; //this line so we don't go into infinite recursion
    doit(std::move(s));
  }
    
}
```
# Return values
unless the compiler performs return value optimization (rvo) the following occurs
(in g++ or clang++ specify -fno-elide-constructors to skip optimization)

```cpp
#include <iostream>
struct Test {
        Test(){
          std::cout<<"ctor\n";
        }
        Test(const Test& rhs){
          std::cout<<"copy ctor\n";
        }
        ~Test(){
          std::cout<<"dtor\n";
        }
};
Test retTest(){
        return Test();
}
int main(){
  Test t=retTest();
}
```
what happens is the following
1. inside function ```retTest()``` an object of type Test is created on the stack
1. a tmp object of type Test is copy constructed from that object
1. the object on the stack is dtored
1. t in main is copy ctored from the tmp
1. tmp is destroyed
1. when main exists t is destroyed

You can test the code below [here](https://godbolt.org/z/9Y4ojx). **Note** the -fno-elide-constructors option in the bottom right of the screen.
```
$g++-10 -fno-elide-constructors -std=c++11 rvopt.cpp
$./a.out
ctor
copy ctor
dtor
copy ctor
dtor
dtor

```
If we remove the -fno-elide-constructors you get this output. Try it [here](https://godbolt.org/z/73bPvr)
```
$g++-10 -std=c++20 rvopt.cpp
$./a.out
ctor
dtor
```

# Pointers
A pointer variable is a variable that holds an address. We say variable _p_ points to variable _x_ if _p_ holds the address of _x_: ```int *p=&x;```.

```cpp
int main(){
int x=17,y=45;
int* p=&x;
std::cout<<p<<std::endl;//prints the value of p, i.e. the address of x
std::cout<<*p<<std::endl;//prints the value store at the location p, i.e. x
*p=23;//change the value of x
p=&y;//p now stores the address of y
}
```

Pointers usually are used when we need to dynamically allocate memory.

```cpp
int main(){
    int *p=new int;//reserve space for int. Value undefined
    int *q=new int(8);//reserve space for int and store 8
    *p=55;//store value 55 at address p
    delete p;//release the reserved memory;
}
```
The **new** operator can be used with any object.

```cpp
#include <iostream>
class Test {
int _x,_y;
public:
 Test(int x,int y):_x(x),_y(y){}
 int& getX(){return _x;}
 int& getY(){return _y;}
};
int main(){
    Test* t=new Test(13,18);
    t->getX()=3;
    t->getY()=7;
    std::cout<<++t->getX()<<"\n";
    std::cout<<++t->getY()<<"\n";
}

```
You can try the above code [here](https://godbolt.org/z/zMach4)

## new, malloc, operator new

A __new__ expression is used both for dynamically allocating memory(on the heap) __and__ calling the constructor of an object. The function __operator new__ allocates memory __only__. In that sense it is similar to malloc in C. Unless you are designing your own container you __almost never__ need to use __operator new__. Usually it is used to _place_ the constructed object at a _preallocated_ place.
Example

```cpp
#include <iostream>
#include <new>
struct Test {
 int _x,_y;
 Test(int x,int y):_x(x),_y(y){std::cout<<"ctor\n";}
 ~Test(){std::cout<<"dtor\n";}
};
int main(){
void* p=operator new(sizeof(Test));
std::cout<<"finished allocating memory\n";
Test* t=new (p) Test{1,2};
delete t;
}
```
You can run the code [here](https://godbolt.org/z/faYq3Y).

We can override the implementation of __operator new__ and __operator delete__. As can be seen below new and delete are the C++ "versions" of C malloc and free.

```cpp
#include <iostream>
class Test {
int _x,_y;
public:
 Test(int x,int y):_x(x),_y(y){std::cout<<"ctor\n";}
 int& getX(){return _x;}
 int& getY(){return _y;}
 ~Test(){std::cout<<"dtor\n";}
};

void * operator new(std::size_t size){
   std::cout<<"allocating size ="<<size<<" \n";
   void *p=malloc(size);
   return p;

}
void operator delete(void *p) noexcept {
    std::cout<<"freeing memory\n";
    free(p);
}
int main(){
    Test *t=new (Test){1,2};
    delete t;
    Test *p=new Test{3,4};
    p->~Test();
    operator delete(p);
  
  
}
```
https://godbolt.org/z/orqKPq

## Smart pointers

While pointers provide flexibility they can cause a variety of problems. One of the problems is shared ownership of a resource. As discussed before, in many situations a pointer variable contains the address of a dynamically allocated memory. We also saw that, in order not to have memory leaks, we need to free the allocated memory when done. This particular problem occurs when two or more pointer variables have the address of the same dynamically allocated resource. In such a situation we either try to free the memory multiple times or access a resource that no longer exists.
We illustrate with the following simple example.

```cpp
#include <iostream>
struct Test {
    int* resource = nullptr;
    Test() {
        resource=new int(10);
        std::cout << "ctor: allocate\n";
    }
    ~Test() {
        if (resource) {
            delete resource;
            std::cout << "deallocate\n";
        }
        std::cout << "dtor\n";
    }
};
int *alloc(){
    return new int(33);
}
int * mod (int *p){
    *p = 14;
    return p;
}

int main() {
    Test* p = new Test();
    int *r=alloc();
    int* t = mod(p->resource);
    //free memory pointed to by r
    delete r;
    //free memory pointed to by p (including resource)
    delete p;

    //if we free memory pointed to by t
    // which doesn't exit the program will crash
    

    std::cout << *t << std::endl;
    //also if we try to access it we get garbage
    delete t;
    std::cout << "done\n";
}

```

To avoid problems like these we use either std::unique_ptr<T> or std::shared_ptr<T>. The first enforces __exclusive__
ownership whereas the second allows __shared__ ownership. We start with an example of the second.

```
#include <iostream>
#include <memory>

struct Test {
   
    std::shared_ptr<int> res;

    Test() {
    
        res = std::make_shared<int>(10);
        std::cout << "ctor: allocate\n";
    }
    ~Test() {
        std::cout << "dtor\n";
    }
};

std::shared_ptr<int> mod(std::shared_ptr<int> p){
    *p = 14;
    return p;
}
int* alloc() {
    return new int(33);
}

int main() {
    Test* p = new Test();
    int* r = alloc();

    std::shared_ptr<int> t = mod1(p->res);
    delete p;
    delete r;
    std::cout << *t << std::endl;
    std::cout << "done\n";
}

```
First, there is no __delete__ of a shared_ptr because it is automatically destroyed (i.e. dtor is called) when it goes out of scope __and__ it frees the resource it is pointing to __only if__ it is the __last__ shared_ptr copy.  Second, its used is exactly like pointers.
In the example above there are two ```std::shared_ptr``` both pointing to the resource allocated by the Test object, namely _p_ indirectly and _t_ directly. The statement ```delete p;``` calls ```~Test()``` which automatically calls the dtors of all members, including _resource_. Since resource is a shared_ptr it knows that it is not the last object pointing to the allocated memory so it does not free it, just decrements the number of copies.

Now when _t_ goes out of scope, it is the last shared_ptr pointing to the resource and therefore it frees it.
You can try both versions [here](https://godbolt.org/z/eMqTMh)

The ```std::unique_ptr<T>``` is similar except it enforces __exclusive__ ownership. Below is the same example using
```std::unique_ptr<T>```. Note the two changes due to noncopiable nature of ```std::unique_ptr```. First, in function ```mod``` the ```std::unique_ptr``` must be passed and returned by reference because we cannot make a copy. Also
in the statement ```    std::unique_ptr<int> t = std::move(mod(p->res));``` the resource held by _p_ is transferred to _t_.
```cpp
#include <iostream>
#include <memory>
struct Test {
    std::unique_ptr<int> res;
    Test() {
        res = std::make_unique<int>(10);
        std::cout << "ctor: allocate\n";
    }
    ~Test() {
        std::cout << "dtor\n";
    }
};
//Note the return as reference since
// a std::unique_ptr cannot be copied
std::unique_ptr<int>& mod(std::unique_ptr<int> & p) {
    * p = 14;
    return p;
}
int* alloc() {
    return new int(33);
}
int main() {
    Test* p = new Test();
    int* r = alloc();
//transfer of ownership using std::move because
// std::unique_ptr cannot be copied
    std::unique_ptr<int> t = std::move(mod(p->res));

    delete p;
    delete r;
    std::cout << *t << std::endl;
    std::cout << "done\n";
}
```
You can try the code [here](https://godbolt.org/z/7c6KT6)
# Templates

On many occasions we write multiple versions of the same code to handle different types. For example suppose we want to write a function to add two numbers (using the + operator) we write

```cpp
int add(int x,int y){
    return x+y;
}
int add(double x,double y){
    return x+y;
}

```
Recall also that the + operator can be used to concatenate strings so we have to add that also. Since the all of those versions only the type changes, c++ allows us to pass the type as a parameters using templates.

```cpp
#include <iostream>
#include <string>

template<typename T>
T add(T x,T y){
    return x+y;
}
int main(){
    int x=2,y=3;
    double u=3.4,v=3;
    std::string s="hello",k="there";

    std::cout<<add(x,y)<<std::endl;
    std::cout<<add(u,v)<<std::endl;
    std::cout<<add(s,k)<<std::endl;
}

```

In the above example the compiler automatically deduces the type which sometimes it cannot and we have to specify it as follows:

```cpp
add<int>(x,y);
add<double(u,v);
add<std::string>(s,k);

```

Note that the template is instantiated __as needed__ at compile time. Also, we can pass parameters to the template other than types. For example

```cpp
template <int n>
void doit(){
    int a[n];
}
```

# Algorithms in STL
The standard template library STL defines a set of general purpose algorithms. You are almost always advised to use those instead of writing your own. Furthermore, since C++17, most of them can take advantage of parallelisation. In this section we look at a few examples. Most of these algorithms need the <algorithm> or <numeric> header and they are defined over a range [start Iterator, end Iterator).

1. std::count and std::count_if ([signature](https://en.cppreference.com/w/cpp/algorithm/count))
```
#include <iostream>
#include <algorithm>
#include <vector>
bool is_even(int x){
    return x%2==0;
}
struct is_odd{
 bool operator()(int x){
     return x%2!=0;
 }
};
int main(){
    std::vector<int> v {7,2,4,2,2,8,2};
    int r=std::count(v.begin(),v.end(),2);
    std::cout<<" number of 2's is "<<r<<"\n";
    int even=std::count_if(v.begin(),v.end(),is_even);
    int odd=std::count_if(v.begin(),v.end(),is_odd());
    std::cout<<" number of even is "<<even<<" and odd is "<<odd<<"\n";
}
```
As you can see count_if takes a unary predicate as a third parameter and this can be  either a function pointer or a function object. Recall, a function object is one that defines an operator().
[click here to run](https://godbolt.org/z/nY35hr)


### Lambda expressions

As we saw, many algorithms, especially in STL, take a __callable__ object as a parameter.  Lambda expression are a convenient way to define such a callable object. We rewrite the previous example using lambda expressions.

```
#include <iostream>
#include <algorithm>
#include <vector>

int main(){
    std::vector<int> v {7,2,4,2,2,8,2};
    int r=std::count(v.begin(),v.end(),2);
    std::cout<<" number of 2's is "<<r<<"\n";
    int even=std::count_if(v.begin(),v.end(),
        [](int x){return x%2==0;});
    int odd=std::count_if(v.begin(),v.end(),
        [](int x){return x%2!=0;});
            std::cout<<" number of even is "<<even<<" and odd is "<<odd<<"\n";
    //counting the number of 2's in a different way
    // just to introduce capture
    int val=2;
    std::count_if(v.begin(),v.end(),
        [val](int x){return x==val;});
}
```
[click here to run](https://godbolt.org/z/MsE1jY)
The brackets "[]" introduces the lambda expression, sometimes called the capture list. The "()" are the parameters passed to the expression, very similar to function arguments. Finally, between "{" and "}" is the body.

Note the __capture__ of the variable _val_. We could have used "[&val]" to capture it by reference. If we want to capture all the variables we use "[=]" and by reference "[&]".


# Extra

## Constructors...

A constructor builds an object bottom up.
1. the constructor of the base class (if any) is called
1. members instructors are called
1. Finally the constructor body is executed.

For example

```cpp
struct Item {
    Item(){std::cout<<"Item ctor\n";}
    Item(int i){std::cout<<"Item ctor with input\n";}
};
struct Test {
    Item _i;
    int x;
};
void noinit(){
    int x=12,y=77,z=99;
    Test t;
    std::cout<<t.x<<std::endl;
}
void init(){
    int x=12,y=77,z=99;
    Test t {};//initialize to zero
    std::cout<<t.x<<std::endl;
}
int main(){
    noinit();
    init();
}
```
Run to get the output
```cpp
item ctor
723520304
item ctor
0
```
As we can see from the above example built-in types are __not__ initiaized: some times they are zero sometimes they are not, it depends on the compiler. For class types the default constructor is called. We can control the constructor and the initialization of members as follows
```cpp
struct Test {
    Test(int x,int i):_x(x),_i(i){}
}
```
