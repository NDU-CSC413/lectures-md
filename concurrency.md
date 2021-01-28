# Multithreading 



# Multithreading in  C++
Starting with the C++11 standard, C++ provides support for writing _portable_ multitheaded applications without relying on additional libraries and extensions. The basic functions and classes for thread support are declared in the ```<thread>``` header. We introduce threads with the ubiquitous hello thread program.
# Creating threads

Starting a new thead in C++ is done by constructing a new ```std::thread``` object. Thread execution begins __immediately__ after the ```thread``` object is constructed. A simple example,

```cpp
#include <iostream>
#include <thread>

// every thread has an "entry" function
void hello() {
    std::cout << "hello thread\n";
}
int main()
{
    std::thread t(hello);
    t.join();
    return 0;
}

```

Thread execution starts at the __entry__ or __top level__ function passed as an argument to the thread object constructor. In the example above the __entry__ or __top level__ function is ```hello()```. 
Since the return value of the __top level__ function is ignore it is usually returns void. 
The Note that the program now has __two__ threads, the initial one running ```main()``` and the one we have created. The call to ```join()``` is important, otherwise the main thread could finish before the thread ```t```.
Note that we can choose ```detach()``` instead of ```join()```. In this case the thread would continue even after the main thread finished. In either case, we need to choose one of the two options, otherwise when the thread destructor is called it terminates our program. Almost in all use cases we use ```join()```.

The example below is an illustration of the use of ```detach```.

```cpp
#include <iostream>
#include <thread>
#include <fstream>

#define WAIT

void write_to_file() {
	std::ofstream output;
	output.open("output.txt");
	output << "first line\n";
	output << "second line\n";
	output.close();
}
int main()
{
	std::thread t{ write_to_file };
	t.detach();
#ifdef WAIT
	std::this_thread::sleep_for(std::chrono::minutes(1));
#endif
}
```
As you can see we use ```detach()``` instead of ```join()```. Therefore the calling thread (in this case main) does not wait for the thread ```t```. But when the main thread exists all other threads are destroyed. Therefore the created thread might not have enough time to perform its job. You can see these two cases by defining and undefining  __WAIT__ in the code.
# Handling exceptions
We close this section by considering the case when an __exception__ is thrown before a thread is joined. Consider the following code

```cpp
#include <iostream>
#include <thread>
#include <exception>
void myf(){
    throw std::exception{};
}
void runt(){
    std::thread t([](){std::cout<<"started thread\n";});
    myf();
    t.join();

}
int main(){
    try {
        runt();
    }
    catch(std::exception e){
        std::cout<<e.what()<<"\n";
    }
    std::cout<<"main thread is done\n";
}
```
Try running this code [here](https://godbolt.org/z/K3a96M).
As you can see the statement ```std::cout<<"main thread is done\n"``` is never reached. This is because when the destructor of a thread object is called (i.e. ~std::thread()) it calls ```std::terminate()``` __if__ the thread is __joinable__. Calling ```std::thread::join()``` or ```std::thread::detach()``` make the thread __not joinable__ and therefore does not call ```std::terminate()```.

In the above example, ```t.join()``` is never reached because ```myf()``` throws an exception. Therefore  ```t.~thread()``` calls ```std::terminate()```.
One way to guard against such a situation to use the _Resource acquisition is initialization_ (RAII) technique. We will see more of this technique when we study mutexes and locks but for now it is sufficient to say that we construct an object wrapper around the thread so that it automatically calls ```join()``` when it is destroyed.

In the code below we define a class ```thread_guard``` that __automatically__ calls ```join``` when its destructor is called. Since the destructor is called when the object goes out of scope this guarantees that the created thread will be joined even when the scope that created the thread takes an unexpected flow due to exceptions.

```cpp
#include <iostream>
#include <exception>
#include <thread>
void threadf() {}
void throwf() {
    std::cout << "starting throwf\n";
    throw std::exception{  };
}
struct thread_guard {
    std::thread& _t;

    thread_guard(std::thread& t) :_t(t) { }
    ~thread_guard() {
        if(_t.joinable())
            _t.join();
        std::cout << "thread guard dtor\n";
    }
};
void runt() {
    std::thread t(threadf);
    thread_guard g(t);
    throwf();
}
int main()
{
    try {
        runt();
    }
    catch (std::exception & e) {
        std::cout<<e.what()<<"\n";
    }
    std::cout << "main thread is done\n";
}
```
You can try the above code [here](https://godbolt.org/z/8MG91T)

Our solution __does not__ guard against the case when an exception occurs __inside__ the thread code. In the previous example if the thread _t_ runs function ```throwf``` instead of ```threadf``` our thread_guard solution does not work. This is because exceptions __are not__ transferred between threads so we need to handle it locally. One solution is to provide an exception safe wrapper as in the example below.

```cpp
#include <iostream>
#include <exception>
#include <thread>
void threadf() {}
void throwf() {
    std::cout << "starting throwf\n";
    throw std::exception{  };
}
struct thread_guard {
    std::thread& _t;

    thread_guard(std::thread& t) :_t(t) { }
    ~thread_guard() {
        if(_t.joinable())
            _t.join();
        std::cout << "thread guard dtor\n";
    }
};
void runt() {
    std::thread t(throwf);
    thread_guard g(t);
    throwf();
}
template<typename F>
void wrapper(F f){
   try{
       f();
   }
   catch(...){}
}
int main()
{
    std::thread t(wrapper<void (*)()>,throwf);
    t.join();
    std::cout << "main thread is done\n";
}
```
You can try the code [here](https://godbolt.org/z/Gqbvvj).

In this case the compiler can not automatically deduce the template parameter type ```F``` because we are calling ```wrapper``` indirectly inside a thread object. Compare that with just calling from main ```wrapper(throwf)``` which the compiler can automatically deduce.

If passing the type of ```throwf``` to the template looks complicated you can use ```decltype```, i.e.
```std::thread t(wrapper<decltype(throwf)>,throwf)```
# Function Objects and Lambdas

In addition to functions, threads can be passed function objects or lambdas as parameters. For example

```cpp
#include <iostream>
#include <thread>
struct bg_task{

   void operator() (){
     std::cout<<"function object \n";
   }
};

int main(){
    std::thread a(bg_task{});
    std::thread b([](){
        std::cout<<"calling lambda\n";
    });
a.join();
b.join();

}
```
You can try this simple example [here](https://godbolt.org/z/b3TcYe). Note that when using gcc we need to pass the pthread switch.

# Passing parameters

Passing arguments to the function is done by passing extra arguments to the thread constructor.
For example,

```cpp
#include <iostream>
#include <thread>
struct fobj{
    void operator()(std::string s){
        std::cout<<s<<"\n";
    }
};
void threadf(std::string s){
    std::cout<<s<<std::endl;
}
int main(){
    int x=0;
    std::thread q(fobj{},"Hello function object");
    std::thread p([](std::string s){
        std::cout<<s<<"\n";
        },"Hello lambda");
    std::thread t(threadf,"hello function");
    p.join();
    q.join();
    t.join();
}
```
You can try the above code [here](https://godbolt.org/z/oTaYxn).

Internally, the thread constructor passes arguments to the callable object as an rvalue. Therefore if the callable object expects the argument to be a reference then use std::ref.

The code below will not compile
```cpp
#include <iostream>
#include <thread>
void threadf(int& x){
    x=17;
}
int main(){
    int x=2;
    std::thread t(threadf,x);
    t.join();
    std::cout<<x<<std::endl;
}
```
But this one does, __note__ the use of ```std::ref```.

```cpp
#include <iostream>
#include <thread>
void threadf(int& x){
    x=17;
}
int main(){
    int x=2;
    std::thread t(threadf,std::ref(x));
    t.join();
    std::cout<<x<<std::endl;
}
```
Try it [here](https://godbolt.org/z/G138To)

A different way to passing parameters by reference would be to use a function object that stores a reference to the variable.
Example

```cpp
#include <thread>
#include <iostream>
struct foo {
	int& _x;
	foo(int& x) :_x(x) {}
	void operator()() {
		_x = 19;
	}
};
int main(){
    int x=8;
    std::thread t(foo{x});
    t.join();
    std::cout<<x<<"\n";
}
```

# Transferring thread ownership

Sometimes we need to transfer ownership of thread. For example, we we need to store threads in containers. But thread objects are __not copyable__. Only one thread object can be associated with a thread of execution at any given time. In these cases we must use __move semantics__.
Here an example to illustrate the problem.


