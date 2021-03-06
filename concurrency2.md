#    Sharing data between threads
  
  
  
  
First note that if the data shared between threads is never modified then we don't have a problem at all. The problems start to occur when one or more threads modifies the shared data.
  
  
##    Motivating Example
  
  
  
In the code below we define two functions: ```void add(int &)``` and ```void sub(int &)``` that increment/decrement the passed parameter by 1.
We create 4 threads: 2 that increment/decrement a variable ```x``` and the other 2 do the same for ```y```.
Since, for each variable, the number of increments is the same as decrements, we expect that, if we initialize to 0, that the final result is also 0. As you can see by running the code this is not the case. __Note__ the large number of iterations we are doing in ```add``` and ```sub```. If we did a single operations we won't catch the problem 
because in that case the calling thread would finish execution before the system switches to another thread.
  
```cpp
#include <iostream>
#include <thread>
#include <vector>
  
#define NUM_ITERATIONS 100000
#define NUM_TRIALS 6
void add(int& val) {
    for (int i = 0; i < NUM_ITERATIONS; ++i) 
        ++val;
}
void sub(int& val) {
    for (int i = 0; i < NUM_ITERATIONS; ++i)
        --val;
}
int y = 0;
  
int main()
{
    for (int j = 0; j < NUM_TRIALS; ++j) {
        std::vector<std::thread > mythreads;
        int x = 0;
        y = 0;
        std::thread t1(add,std::ref(x));
        std::thread t2(add, std::ref(y));               
        mythreads.push_back(std::move(t1));
        mythreads.push_back(std::move(t2));
      /* using temp thread object no need for std::move*/
        mythreads.push_back(std::thread(sub, std::ref(x)));
        mythreads.push_back(std::thread(sub,std::ref(y)));
        for (auto& t : mythreads)
            t.join();
  
       std::cout << "trial " << j << ",x=" << x << ",y=" << y << std::endl;
    }
}
```
You can try a __modified__ version of the above with only one variable and two threads 
but a __larger__ number of iterations [here](https://godbolt.org/z/7jjW91 ). Compiler explorer
doesn't allow us to create more than two threads.
  
###  Analysis
  
Consider the assembly code produced by gcc (slightly changed by me) of the ```add``` function 
use in the previous code.  The operation ```++var;``` is basically translated into three steps
1. load the value of _var_ into register _edx_. We will call it ```mov var edx```.
1. increment register _edx_. We will call it ```inc edx``` or ```dec edx``` if it is decremented.
1. store the value in _edx_ back to _var_. We will call it ```mov edx var```;
  
  
  
```assembly
  
add:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -24(%rbp)   #store the address of val on the stack
        movl    $0, -4(%rbp)
        movq    -24(%rbp), %rax   #copy the address of val to rax
.L3:
        cmpl    $99999, -4(%rbp) #compare with index with 99999
        jg      .L4              #if index > 99999 goto L4
=>        movl    (%rax), %edx     #copy the value of val to edx
=>        inc     %edx             #increment edx
=>        movl    %edx, (%rax)     #set the value of val to value in edx
        addl    $1, -4(%rbp)
        jmp     .L3
.L4:
        nop
        popq    %rbp
        ret
main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $16, %rsp
        movl    $7, -4(%rbp)
        leaq    -4(%rbp), %rax
        movq    %rax, %rdi    #move the address 7 to rdi
        call    add
        movl    $0, %eax
        leave
        ret
```
  
  
  
Now consider, say, two threads, one executing ```++var``` and the other ```--var``` with ```var``` initially,
equal to 4. Since the two threads are independent there are many possible sequences of instructions,
__one__ of them is shown below. As you can see, the final value of ```var``` in this case is 3. Since other 
sequences are possible, one could get 3,4, or 5 depending which thread finishes first. What is worse, if
the two threads are executing in a long loop, for every iteration we might get a different value. This is called  a
__race condition__ and the part of the code where the shared variable is accesseed is called __critical section__. That is why in the previous code we got non consistent results.
  
```
mov var edx1    # edx1=4
mov var edx2    # edx2=4
inc edx1        # edx1=5
dec edx2        # edx2=3
mov edx1 var    # var=5
mov edx2 var    # var=3
```
##   std::mutex
  
  
One way to synchronize access to a __critical section__ is to use ```std::mutex```. Essentially, ```std::mutex```
behaves as a lock. We modify the previous code as follows:
```cpp
#include <mutex>
std::mutex m;
 void add(int& val) {
        m.lock();
        for (int i = 0; i < NUM_ITERATIONS; ++i) 
                ++val;
        m.unlock();
}
void sub(int& val) {
        m.lock();
        for (int i = 0; i < NUM_ITERATIONS; ++i)
                --val;
        m.unlock();
}
```
  
 You can try the complete code [here](https://godbolt.org/z/9Kab8x ) 
  
##  std::lock_guard
  
Sometimes the above approach leads to problems since the ```std::mutex``` is not unlocked _automatically_. When the code base gets complicated, a programmer might simply forget to unlock a mutex and a deadlock occurs. More importantly, suppose that before we call ```std::mutex::unlock()``` a exception occurs. This means that the unlock statement is never reached.
Consider the code below.
```cpp
#include  <string>
#include <random>
std::random_device rd;
std::uniform_int_distribution<> dist(50, 500);
  
std::mutex m;
int add(const std::vector<std::string>& values) {
    int val = 0;
    std::this_thread::sleep_for(std::chrono::milliseconds(dist(rd)));
    m.lock();
    for (int i = 0; i < values.size(); ++i)
        val += std::stoi(values[i]);
    m.unlock();
    return val;
}
int sub(const std::vector<std::string>& values) {
    int val = 0;
    /* wait twice as much as add on average */
    std::this_thread::sleep_for(std::chrono::milliseconds(2*dist(rd)));
    m.lock();
    for (int i = 0; i < values.size(); ++i)
        val -= std::stoi(values[i]);
    m.unlock();
    return val;
}
  
  
template <typename T>
void threadf(int& val, const T& values
    , int (*f)(const T&)) {
    /* we MUST catch exception here.
    * it cannot be caught by main
    */
    try {
        val += f(values);
    }
    catch (std::exception& e) {
        std::cout << e.what() << "\n";
    }
}
```
The functions ```add``` and ```sub``` will be called from two different threads. Each locks mutex ```m``` before 
performing their operations, which is to convert a set of strings to integers and add/sub them to the passed variable.
The ```sleep_for``` is added such that ```add``` locks the mutex first, most of the time.
The ```std::stoi``` will raise an exception if the input string cannot be converted to an integer.

Below is the ```main``` function. Notice that the input to ```add``` will be the vector ```evalues``` which contains
the character _p_ as a first value. This will cause an exception in ```add``` which will be caught in ```threadf```,
causing ```sub``` to hang since it is waiting for the release of  a locked mutex that will not be released.
__NOTE__ an exception raised in a thread __cannot__ be caught by the parent thread.

    
```cpp
int main()
{
    using V = std::vector<std::string>;
    int sum = 0;
    V values{ "1","2","3","4","5" };
    V evalues{ "p","2","3","4","5" };//causes exception in std::stoi
    /* due to exception this thread will exit without releasing mutex*/
    std::thread t1(threadf<V>, std::ref(sum), std::ref(evalues), add);
    /* this one will most probably hang waiting for the mutex to be released*/
    std::thread t2(threadf<V>, std::ref(sum), std::ref(values), sub);
    t1.join();
    t2.join();
  
    std::cout << sum << "\n";
}
```
 For these reasons C++ supplies us with a convenient object ```std::lock_guard<std::mutex>``` which is basically
a wrapper object around a ```std::mutex```. It follows the _resource acquisition is initialization_ [(RAII)](https://en.cppreference.com/w/cpp/language/raii) idiom.
This means that the mutex is locked/unlocked when the object is created/destroyed. Using ```std::lock_guard``` is easy.
Simply replace ```std::mutex m;``` with ```std::lock_guard<std::mutex> g(m);``` and remove ```m.unlock```. When the 
```lock_guard``` object is created, it locks the mutex, and when it is destroyed it unlocks it automatically.
```cpp
int add(const std::vector<std::string>& values) {
    int val = 0;
    std::this_thread::sleep_for(std::chrono::milliseconds(dist(rd)));
    std::lock_guard<std::mutex> g(m);
    for (int i = 0; i < values.size(); ++i)
        val += std::stoi(values[i]);
    return val;
}
```
You can try the lock_guard version [here](https://godbolt.org/z/efoabb)

## Readers-Writers

In some situations, a simple locking/unlocking is not satisfactory. This is typically the case when some threads
__do not__ modify the shared data as in the classical case of Readers-Writers. In this scenario we can have two 
types of threads: readers and writers. As the name implies the readers threads do not modify the shared data 
whereas the writers do. Simply using a lock/unlock mechanism will prevent multiple readers from reading 
"at the same time" , which speeds things up, even though there is no harm in doing so. Before proceeding
let us review the constraints on the threads
1. A reader can access the shared data even if other readers are doing so
1. A reader cannot access the shared data if a writer is doing so
1. A writer cannot access the data if __any__ thread is doing so.

The classical solution is to have the __first__ reader lock a mutex and the __last__ one unlocks it.
To be able to implement the above logic we need to resolve the following two issues
1. How to keep track of the first/last reader thread. This is done by keeping a counter
of the number of reader threads. 
2. We cannot unlock a ```std::mutex``` that was locked by __another thread__.

Both of these problems are solved using  ```std::shared_mutex```.

A ```std::shared_mutex``` has two levels of access: __exclusive__, only one thread can own the mutex, and __shared__,
multiple threads can share the ownership.


```cpp
/*****************************************************************//**
 * @file   concurrency2.3.cpp
 * @brief  Readers-Writers using std::shared_mutex
 * 
 * @author Hikmat Farhat
 * @date   February 2021
 *********************************************************************/

#include <iostream>
#include <vector>
#include <random>
#include <thread>
#include <mutex>
#include <shared_mutex>

std::random_device e;
std::uniform_int_distribution<> dist(1, 50);
std::vector<int> v;// shared buffer

std::shared_mutex wrt;

class Reader {
public:
    void operator() () {
        int sum = 0;
        std::this_thread::sleep_for(std::chrono::milliseconds(dist(e) * 100));        
        wrt.lock_shared();
        std::cout << "Reader thread " << std::this_thread::get_id() << " started\n";
        for (auto x : v) {
            sum += x;
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
        std::cout << " Reader thread " << std::this_thread::get_id() << " ended\n";
        wrt.unlock_shared();
        if (sum != 0) std::cout << "sum in thread " << std::this_thread::get_id() << " is " << sum << std::endl;
    }
};

class Writer {

public:
    Writer() {
    }
    void operator() () {
        int value = dist(e);
        std::this_thread::sleep_for(std::chrono::milliseconds(dist(e) * 100));
        wrt.lock();
        std::cout << " Writer thread " << std::this_thread::get_id() << " started\n";

        for (auto& x : v) {
            x = value;
            value = -value;
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }
        std::cout << " Writer thread " << std::this_thread::get_id() << " ended\n";
        wrt.unlock();

    }
};

int main()
{
    std::cout << "Concurrency 2.3 \n";
    std::cout << "------------------------------\n";
    int value = 1;
    for (int i = 0; i < 100; i++) {
        v.push_back(value);
        value = -value;
    }
    for (int i = 0; i < 4; i++) {
        std::cout << "Trial " << i << std::endl;
        std::vector<std::thread> mythreads;
        for (int i = 0; i < 2; i++) {
            Reader r1, r2;
            Writer w;
            std::thread t1(r1);
            std::thread t2(w);
            std::thread t3(r2);
            mythreads.push_back(std::move(t1));
            mythreads.push_back(std::move(t2));
            mythreads.push_back(std::move(t3));
        }
       
        for (auto& t : mythreads)
            t.join();
        std::cout << "----------------" << std::endl;
        
    }

}


```
  
  
## std::unique_lock
  
## Atomics
  
## Thread safe data structures
  
## Barriers
  
## Parallel mergesort
  
  
  
  
  