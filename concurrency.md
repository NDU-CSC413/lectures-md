Consider the code below


```
#include <iostream>
#include <thread>


// every thread has an "entry" function
void hello() {
    std::cout << "hello concurrency\n";
}
int main()
{
    std::thread t(hello);
    t.join();
}

```

Each thread must have a entry function. In the example above the entry is the function hello(). Note that the program now has 2 threads, the initial one running main and the one we have created. The call to join() is important, otherwise the main thread could finish before the thread t.
Note that we can choose detach() instead of join(). In this case the thread would continue even after the main thread finished. In either case, we need to choose one of the two options, otherwise when the thread destructor is called it terminates our program. Almost in all use cases we use join().

## Function Objects

Threads can be passed function objects as parameters. For example

```
class bg_task{
private:
int x;
bg_task(int v):x(v){}
void operator() (){
 std::cout<<"value of x is "<<x<<"\n";

}

};

```

## Passing parameters

Passing arguments to the function or callable object is done by passing extra arguments to the thread constructor.
For example,

```
void threadf(std::string s){
    std::cout<<s<<std::endl;
}
int main(){
    std::thread t(threadf,"hello thread");
    t.join();
}

```
Internally, the thread constructor passes arguments to the callable object as an rvalue. Therefore if the callable object expects the arugment to be a reference then use std::ref.

The code below will not compile
```
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

But this one does, note the use of std::ref.

```
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

## Transferring thread ownership

Thread objects are moveable but not copyable. Only one thread object can be associated with a thread of execution at any given time.

