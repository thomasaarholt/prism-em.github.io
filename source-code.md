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

I built a custom multidimensional array class for *PRISM*, mainly because I wanted to have full control over how the data was represented to get the best performance (i.e. make sure the data is always contiguous and use raw pointers when I could get away with it). The contiguous bit is super important. You could choose to represent multidimensional arrays in C++ as pointers-to-pointers or pointers-to-pointers-to-pointers, etc, which allows for enticing syntax like `data[0][0]`; however, this method allocates more memory and almost certainly allocates it discontiguously. This would totally kill your performance by causing each data access to invoke multiple pointer dereferences, requiring more operations and causing cache misses all over the place. Even if you use a clever trick to make repeated syntax bracket reduce to a single index, it generates unnecessary function call overhead. My solution was to instead store the data internally in a 1D buffer and then access it with a `.at()` member function that is aware of what the "actual" dimensionality of the `PRISM::Array` is supposed to be. Because the last dimension changes the fastest in C-style ordering, I chose the syntax of the `at()` method to be slowest-to-fastest indices as you read left-to-right. For example that is `.at(y,x)` for a 2D array, `.at(z,y,x)` for a 3D array, etc. By choosing `std::vector` to hold that 1D data buffer, I don't have to worry about `new`/`delete` or any other dynamic allocation business and garbage collection. Whenever I need to loop over the whole array (common), I also implemented the typical `begin()` and `end()` methods, which conveniently also allow for [range-based for loops](http://en.cppreference.com/w/cpp/language/range-for) with modern C++. Many operations in *PRISM* are loops operating on and incrementing raw pointers.

I also added some convenience functions very similar to MATLAB's `zeros` and `ones`.. this was mainly to make my life easier when transcribing from MATLAB code.

The full class can be found in "ArrayND.h" and as it is some 600 lines that are fairly repetetive I won't include the whole thing here. But as an example

~~~ c++
namespace PRISM {
    template <size_t N, class T>
    class ArrayND {
            // ND array class for data indexed as C-style, i.e. arr.at(k,j,i) where i is the fastest varying index
            // and k is the slowest

            // T is expected to be a std::vector
        public:
            ArrayND(T _data,
                    std::array<size_t, N> _dims);
            ArrayND(){};
			size_t get_dimi() const {return this->dims[N-1];}
			size_t get_dimj() const {return this->dims[N-2];}
			size_t get_dimk() const {return this->dims[N-3];}
			size_t get_diml() const {return this->dims[N-4]; }
			size_t get_dimm() const {return this->dims[N-5]; }
            size_t size()     const {return this->arr_size;}
            typename T::iterator begin();
            typename T::iterator end();
            typename T::const_iterator begin() const;
            typename T::const_iterator end()   const;
            typename T::value_type& at(const size_t& i);
            typename T::value_type& at(const size_t& j, const size_t& i);
            ///... many more declarations
            ///...
        
        private:
            std::array<size_t, N> dims;
            std::array<size_t, N-1> strides;
            size_t arr_size;
            T data;
    }
    
    // ... an example .at() implementation late in the filer
      template <size_t N, class T>
    typename T::value_type& ArrayND<N, T>::at(const size_t& k, const size_t& j,const size_t& i){
        return data[k*strides[0] + j*strides[1] + i];
    }
~~~

So the template parameter `T` represents the underlying buffer data type, which must behave as `std::vector`. The dimensions and array strides are stored in fixed arrays, and the `.at()` methods use these strides to compute offsets, such as in the example above for the 2D array case.

I say "behave as" for the data buffer, because originally I also intended this class to be able to contain `thrust::host_vector` and `thrust::device_vector`. This would allow one to effectively use one template class to wrap multidimensional arrays that can transfer data back and force from the GPU without using the low level CUDA API calls. For example, you could overload the `=` operator for arrays of two different types and have the underying vector types copy from one another, also with the `=` operator. For example, `PRISM::ArrayND<T>` and `PRISM::ArrayND<U>` where `T` was a `thrust::host_vector` and `U` is a `thrust::device_vector` would invoke the assignment of a `thrust::host_vector` from a `thrust::device_vector`, calling `cudaMemcpy` under the hood, and I would never have to touch that. The same class simultaneously could be used to assign one `PRISM::ArrayND<std::vector>` to another. All of the metadata about the dimensions, etc, are stored host-side, so in principle this template class would allow you to use one syntax across all your host/device arrays and not see many cuda device calls at all. I have also written about this topic before with the approach of template specialization -- you can read about that [here](http://alanpryorjr.com/image/Flexible-CUDA/).

I still think this is a very good design pattern, but I ended up not using `thrust` at all, and the main reason was because you must use page-locked memory for asynchronous transfers. There is experimental support for a pinned memory allocator in thrust, `thrust::system::cuda::experimental::pinned_allocator< T >`, which can be passed into `thrust::host_vector`. However, with it being an experimental feature I was concerned about stability and figured if I was going to the trouble of customizing my array class specifically for performance I might as well also manually do my own allocations and not take risks. So `PRISM::ArrayND< std::vector<T> >` is used for the pageable host-side arrays, and then raw pointer are used for pinned memory and the device arrays.

### Metadata

The primary way the simulation is setup is through the `Metadata` class. Simulation parameters that the user adjusts exist inside `Metadata` such as the electron energy, algorithm, slice thickness, PRISM interpolation factors, pixel size, etc. The metadata is then used to configure and execute the simulation. The nice thing about arranging the program in this way is that once you have a `Metadata` object it uniquely determines how to proceed with the rest of the simulation. The CLI and the GUI are just two different ways to adjust the parameters in the `Metadata`, but beyond that the rest of the code is the same. You can find the details of the `Metadata` template class in "meta.h" -- it's just a container for a number of variables. Yes, they are public instead of being private with setters/getters, but the only way these metadata values are changed is through higher level methods in the CLI and GUI, so in my opinion being "correct" would just introduce a bunch of extra unnecessary code.

### Command Line Interface

The CLI program, `prism`, is very simple.

~~~ c++
int main(int argc, const char** argv) {
	PRISM::Metadata<PRISM_FLOAT_PRECISION> meta;

	// parse command line options
	if (!PRISM::parseInputs(meta, argc, &argv))return 1;

	// print metadata
    meta.toString();

	// configure simulation behavior
	PRISM::configure(meta);

	// execute simulation
	PRISM::execute_plan(meta);
	return 0;
}
~~~

The various command line options are parsed by the function `parseInputs`, which returns `true` if successful, otherwise the program exits. From there the configuration step is run (more on that later), and the calculation executed.

My take on a command line parser is simple. It uses a `std::map` to connect command argument keywords with functions that handle that particular argument. These functions all have the same signature. Using `std::map` allows for better lookup speed than a switch statement, but more importantly it makes it very easy to connect multiple keywords with the same parsing function. For example, I might have

~~~ c++
    using parseFunction = bool (*)(Metadata<PRISM_FLOAT_PRECISION>& meta,
                                          int& argc, const char*** argv);
    static std::map<std::string, parseFunction> parser{
            {"--input-file", parse_i}, {"-i", parse_i},
            {"--interp-factor", parse_f}, {"-f", parse_f}
            // ... more values follow
~~~ 

So the variable `parser` connects both the verbose and shorthand keywords "--interp-factor" and "-f" to the function `parse_f`, which is the function responsible for populating the PRISM interpolation factor in the metadata.

~~~ c++
    bool parse_f(Metadata<PRISM_FLOAT_PRECISION>& meta,
                        int& argc, const char*** argv){
        if (argc < 2){
            cout << "No interpolation factor provided for -f (syntax is -f interpolation_factor)\n";
            return false;
        }
        if ( (meta.interpolationFactorX = atoi((*argv)[1])) == 0){
            cout << "Invalid value \"" << (*argv)[1] << "\" provided for PRISM interpolation factors (syntax is -f interpolation_factor)\n";
            return false;
        }
	    meta.interpolationFactorY = meta.interpolationFactorX;
        argc-=2;
        argv[0]+=2;
        return true;
    };
~~~

I check that there are at least two remaining arguments, otherwise there isn't a factor provided and `false` is returned. Then the attempt is made to convert the factor to a number. If that fails we return `false`. Otherwise the parsing was successful, the value is set, and `argc` decremented and `argv` shifted. Some parsing functions might require more or fewer arguments, and with this design that's no problem because each option has a separate function. It also makes it very easy to add new options -- I just write the logic for parsing that argument, and then connect whatever keywords with the function name.

### Configuration

Several places in *PRISM* there are divergences in how the simulation proceeds, such as whether to perform the calculation with multislice or PRISM. Rather than constantly having if-then-else statements for this type of logic, I'll instead create a single function pointer that is set to point to the desired behavior by the `configure` function. For example, both PRISM and multislice have entrypoint functions, cleverly named `PRISM_entry` and `Multislice_entry`. There is as corresponding function pointer defined in "configure.h" that is the `execute_plan` function used earlier in the driver:

~~~ c+++
//configure.h
using entry_func = Parameters<PRISM_FLOAT_PRECISION>  (*)(Metadata<PRISM_FLOAT_PRECISION>&);
entry_func execute_plan;
~~~

and this is set in "configure.cpp"

~~~ c++
//configure.cpp
		if (meta.algorithm == Algorithm::PRISM) {
			fill_Scompact = fill_Scompact_CPUOnly;
			//...
			// bunch of other stuff..
			// ...
		} else if (meta.algorithm == Algorithm::Multislice){	
			execute_plan = Multislice_entry;
		}
~~~

If you think this is overkill, and I could just have an if-else for this case, consider that there are choices of PRISM/Multislice, the possibility of CPU-only or GPU-enabled, the possibility of streaming/singlexfer if we are using the GPU codes, etc. It would create a lot of divergences very quickly, and this is a better solution.
