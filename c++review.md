---
title: C++ Review 
---

## Variables and references

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


You can run the above code [here](https://repl.it/@hfarhat/lvalue-rvalue-references).
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
You can run the above code [here](https://repl.it/@hfarhat/lvalue-rvalue-reference2)
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
## Return values
unless the compiler performs return value optimization (rvo) the following occurs
(in g++ or clang++ specify -fno-elide-constructors to skip optimization)

```cpp
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
```cpp
$g++-10 -fno-elide-constructors -std=c++11 rvopt.cpp
$./a.out
ctor
copy ctor
dtor
copy ctor
dtor
dtor
$g++-10 -std=c++20 rvopt.cpp
$./a.out
ctor
dtor
```

## Pointers
A pointer variable is a variable that holds and address. We say variable _p_ points to variable _x_ if _p_ holds the address of _x_: ```int *p=&x;```.

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
## Templates

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
