#  Sharing data between threads
  
  
First note that if the data shared between threads is never modified then we don't have a problem at all. The problems start to occur when one or more threads modifies the shared data.
<p align="center"><img src="https://latex.codecogs.com/png.latex?&#x5C;sqrt(&#x5C;pi)"/></p>  
  
  
##  Motivating Example
  
  
In the code below we define two functions: ```void add(int &)``` and ```void sub(int &)``` that increment/decrement the passed parameter by 1.
We create 4 threads: 2 that increment/decrement a variable ```x``` and the other 2 do the same for ```y```.
Since, for each variable, the number of increment is the same as decrement, we expect that, if we initialize to 0, that the final result is also 0. As you can see by running the code this is not the case. __Note__ the large number of iterations we are doing in ```add``` and ```sub```. If we did a single operations we won't catch the problem 
because in that case the calling thread would finish execution before the system switches to another thread.
  
```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex>
  
#define NUM_ITERATIONS 1000000
#define NUM_TRIALS 10
void add(int& val) {
    for (int i = 0; i < NUM_ITERATIONS; ++i) ++val;
}
void sub(int& val) {
    for (int i = 0; i < NUM_ITERATIONS; ++i) --val;
}
  
int main()
{
    for (int j = 0; j < NUM_TRIALS; ++j) {
        int x = 0;
        std::vector<std::thread > mythreads;
        mythreads.push_back(std::thread(add,std::ref(x)));
        mythreads.push_back(std::thread(sub, std::ref(x)));
        for (auto& t : mythreads)
            t.join();
  
        std::cout << "trial " << j << ",x=" << x << std::endl;
    }
}
```
##  Using std::mutex
  
  
  