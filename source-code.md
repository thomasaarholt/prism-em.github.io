# PRISM Source Code Walkthrough

Just because a project is open source doesn't mean that it is immediately obvious how all of its moving parts fit together.  I've spent a lot of time thinking about *PRISM* and how to design it from a software engineering perspective, and I'm fairly satisfied with the result. This document is meant to be a sort of tour through the source code of *PRISM* in a much more casual environment than a formal academic paper. This is not a tutorial for how to use the code (that is [here](www.example.com)), but rather an explanation of the code itself. Thus it is more focused towards developers and less so to most users. For those of you who haven't frantically closed the browser at this point, I hope that you find it useful either as a GPU programming example, or as a guide to become a future developer of *PRISM* itself.


### Work Dispatcher

The `WorkDispatcher` class is critical to how parallel work is performed in *PRISM*. Conceptually the idea is that there is a bunch of tasks to be done, and there exist a number of worker threads to do these tasks. At this level we don't really care how the worker is doing the work as long as it gets done. So the `WorkDispatcher` exists to hand out tasks to the workers in a synchronized manner, and is extremely simple to implement using `std::mutex`.

~~~ c++
// WorkDispatcher.h
#ifndef PRISM_WORKDISPATCHER_H
#define PRISM_WORKDISPATCHER_H
#include "params.h"
#include "configure.h"
#include <mutex>
namespace PRISM {
    class WorkDispatcher {
    public:
        WorkDispatcher(size_t _current,
                       size_t _stop);

        bool getWork(size_t& job_start, size_t& job_stop, size_t num_requested=1, size_t cpu_early_stop=SIZE_MAX);
    private:
        std::mutex lock;
        size_t current, stop;
    };
}
#endif //PRISM_WORKDISPATCHER_H
~~~

Whenever we end up at a point in the calculation that we want to consume in parallel, you construct a single `WorkDispatcher` that holds a `mutex` and has a starting and stopping index representing all of the work. These indices might map to slices of the projected potential, for example. A reference to this work dispatcher is passed to all of the worker threads, who then request jobs with the `getWork` method. The implementation follows

~~~ c++
#include "WorkDispatcher.h"
#include <mutex>
// helper function for dispatching work

namespace PRISM {
        WorkDispatcher::WorkDispatcher(size_t _current,
					   size_t _stop) :
					   current(_current),
					   stop(_stop){};

		bool WorkDispatcher::getWork(size_t& job_start, size_t& job_stop, size_t num_requested, size_t early_cpu_stop){
			std::lock_guard<std::mutex> gatekeeper(lock);
			if (job_start >= stop | current>=early_cpu_stop) return false; // all jobs done, terminate
			job_start = current;
			job_stop = std::min(stop, current + num_requested);
			current = job_stop;
			return true;
		}
}
~~~

The constructor just initializes the relevant fields with the input parameters. The `getWork` function locks the mutex with a `std::lock_guard` in [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) style such that it is guaranteed to be unlocked if anything bad happens. Everything after the creation of the `lock_guard` is synchronized, and the `WorkDispatcher` then tries to set `job_start` and `job_stop` to a range of `num_requested` jobs if possible, and fewer if necessary. It's really that simple.

### Multidimensional Arrays in PRISM 

I built a custom multidimensional array class for *PRISM*, mainly because I wanted to have full control over how the data was represented to get the best performance (i.e. make sure the data is always contiguous and use raw pointers when I could get away with it). The contiguous bit is super important. You can choose to represent multidimensional arrays in C++ as pointers-to-pointers or pointers-to-pointers-to-pointers, etc, which allows for enticing syntax like `data[0][0]`; however, this method allocates more memory and almost certainly allocates it discontiguously. This would totally kill your performance by causing each data access to invoke multiple pointer dereferences, requiring more operations and causing cache misses all over the place. My solution was to instead store the data internally in a 1D buffer and then access it with a `.at()` member function that is aware of what the "actual" dimensionality of the `PRISM::Array` is supposed to be. Because the last dimension changes the fastest in C-style ordering, I chose the syntax of the `at()` method to be slowest-to-fastest indices as you read left-to-right. For example that is `.at(y,x)` for a 2D array, `.at(z,y,x)` for a 3D array, etc. By choosing `std::vector` to hold that 1D data buffer, I don't have to worry about `new`/`delete` or any other dynamic allocation business and garbage collection. Whenever I need to loop over the whole array (common), I also implemented the typical `begin()` and `end()` methods, which conveniently also allow for [range-based for loops](http://en.cppreference.com/w/cpp/language/range-for) with modern C++.

I also added some convenience functions very similar to MATLAB's `zeros` and `ones`.. this was mainly to make my life easier when transcribing from MATLAB code.