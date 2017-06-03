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

If you think this is overkill, and I could just have an if-else for this case, consider that there are choice of PRISM/Multislice, the possibility of CPU-only or GPU-enabled, the possibility of streaming/singlexfer if we are using the GPU codes, etc. It would create a lot of divergences very quickly, and this is a better solution.

## PRISM

The `PRISM_entry` function is the top-tier control function for PRISM calculations, and handles running each of the steps and computing the various frozen phonon configurations. PRISM is divided into 3 steps:

1. Compute projected potentials
2. Compute compact S-matrix
3. Compute final output

During each of these steps, a number of different data structures are created and modified. To organize these internally used arrays, a `Parameters` class is used. So we now have two wrapping classes: `Metadata`, which contains the user-adjustable parameters, and `Parameters`, which contains the internally modified data structures. This prevents having to juggle around a lot of parameters for each function call, and makes things easier to read.

~~~ c++
#include "PRISM_entry.h"
#include <iostream>
#include <stdlib.h>
#include <algorithm>
#include "configure.h"
#include "ArrayND.h"
#include "PRISM01_calcPotential.h"
#include "PRISM02_calcSMatrix.h"
#include "PRISM03_calcOutput.h"
#include "params.h"
#include <vector>

namespace PRISM{
	using namespace std;
	Parameters<PRISM_FLOAT_PRECISION> PRISM_entry(Metadata<PRISM_FLOAT_PRECISION>& meta){
		Parameters<PRISM_FLOAT_PRECISION> prism_pars;
		try { // read in atomic coordinates
			prism_pars = Parameters<PRISM_FLOAT_PRECISION>(meta);
		} catch(const std::runtime_error &e){
			std::cout << "Terminating" << std::endl;
			exit(1);
		}

		// compute projected potentials
		PRISM01_calcPotential(prism_pars);

		// compute compact S-matrix
		PRISM02_calcSMatrix(prism_pars);

		// compute final output
		PRISM03_calcOutput(prism_pars);

		// calculate remaining frozen phonon configurations
        if (prism_pars.meta.numFP > 1) {
            // run the rest of the frozen phonons
            Array3D<PRISM_FLOAT_PRECISION> net_output(prism_pars.output);
            for (auto fp_num = 1; fp_num < prism_pars.meta.numFP; ++fp_num){
	            meta.random_seed = rand() % 100000;
				Parameters<PRISM_FLOAT_PRECISION> prism_pars(meta);
                cout << "Frozen Phonon #" << fp_num << endl;
	            prism_pars.meta.toString();
	        	PRISM01_calcPotential(prism_pars);
	        	PRISM02_calcSMatrix(prism_pars);
                PRISM03_calcOutput(prism_pars);
                net_output += prism_pars.output;
            }
            // divide to take average
            for (auto&i:net_output) i/=prism_pars.meta.numFP;
	        prism_pars.output = net_output;
        }
        if (prism_pars.meta.save3DOutput)prism_pars.output.toMRC_f(prism_pars.meta.filename_output.c_str());

		if (prism_pars.meta.save2DOutput) {
			size_t lower = std::max((size_t)0, (size_t)(prism_pars.meta.integration_angle_min / prism_pars.meta.detector_angle_step));
			size_t upper = std::min((size_t)prism_pars.detectorAngles.size(), (size_t)(prism_pars.meta.integration_angle_max / prism_pars.meta.detector_angle_step));
			Array2D<PRISM_FLOAT_PRECISION> prism_image;
			prism_image = zeros_ND<2, PRISM_FLOAT_PRECISION>(
					{{prism_pars.output.get_dimk(), prism_pars.output.get_dimj()}});
			for (auto y = 0; y < prism_pars.output.get_dimk(); ++y) {
				for (auto x = 0; x < prism_pars.output.get_dimj(); ++x) {
					for (auto b = lower; b < upper; ++b) {
						prism_image.at(y, x) += prism_pars.output.at(y, x, b);
					}
				}
			}
			std::string image_filename = std::string("prism_2Doutput_") + prism_pars.meta.filename_output;
			prism_image.toMRC_f(image_filename.c_str());
		}

        std::cout << "PRISM Calculation complete.\n" << std::endl;
		return prism_pars;
	}
}
~~~

The `Parameters` object is created (it takes the metadata as one the parameters to its constructor). Next an attempt is made to read the atomic coordinates, and if that fails then the program terminates. If the atoms are successfully read, we then execute each of the three steps. Last we handle the remaining frozen phonons, average the results, and save the output.

## Calculating Projected Potentials

Now we assemble the projected potentials by computing the digitized potential for each atomic species and storing it in a lookup table. The full sliced and projected potential is then calculated by dividing the model into slices and using this lookup table to build each slice.

#### Assembling the lookup table

~~~c++
	void PRISM01_calcPotential(Parameters<PRISM_FLOAT_PRECISION>& pars){
		//builds projected, sliced potential

		// setup some coordinates
		cout << "Entering PRISM01_calcPotential" << endl;
		PRISM_FLOAT_PRECISION yleng = std::ceil(pars.meta.potBound / pars.pixelSize[0]);
		PRISM_FLOAT_PRECISION xleng = std::ceil(pars.meta.potBound / pars.pixelSize[1]);
		ArrayND<1, vector<long> > xvec(vector<long>(2*(size_t)xleng + 1, 0),{{2*(size_t)xleng + 1}});
		ArrayND<1, vector<long> > yvec(vector<long>(2*(size_t)yleng + 1, 0),{{2*(size_t)yleng + 1}});
		{
			PRISM_FLOAT_PRECISION tmpx = -xleng;
			PRISM_FLOAT_PRECISION tmpy = -yleng;
			for (auto &i : xvec)i = tmpx++;
			for (auto &j : yvec)j = tmpy++;
		}
		Array1D<PRISM_FLOAT_PRECISION> xr(vector<PRISM_FLOAT_PRECISION>(2*(size_t)xleng + 1, 0),{{2*(size_t)xleng + 1}});
		Array1D<PRISM_FLOAT_PRECISION> yr(vector<PRISM_FLOAT_PRECISION>(2*(size_t)yleng + 1, 0),{{2*(size_t)yleng + 1}});
		for (auto i=0; i < xr.size(); ++i)xr[i] = (PRISM_FLOAT_PRECISION)xvec[i] * pars.pixelSize[1];
		for (auto j=0; j < yr.size(); ++j)yr[j] = (PRISM_FLOAT_PRECISION)yvec[j] * pars.pixelSize[0];

		vector<size_t> unique_species = get_unique_atomic_species(pars);

		// initialize the lookup table
		Array3D<PRISM_FLOAT_PRECISION> potentialLookup = zeros_ND<3, PRISM_FLOAT_PRECISION>({{unique_species.size(), 2*(size_t)yleng + 1, 2*(size_t)xleng + 1}});
		
		// precompute the unique potentials
		fetch_potentials(potentialLookup, unique_species, xr, yr);

		// populate the slices with the projected potentials
		generateProjectedPotentials(pars, potentialLookup, unique_species, xvec, yvec);
	}
~~~

`get_unique_atomic_species` just determines the minimal set of elements in the sample -- no point in precomputing the potential for atoms we don't have. `fetch_potentials` builds the lookup table based upon the relevant coordinates, which are determined by the pixel size and `potBound`, and then `generateProjectedPotentials` builds the full projected/sliced potential.

Next is a big chunk of code that is conceptually accomplishing a simple task. Based on the relevant coordinates determined by the simulation pixel size, we compute the potential of each relevant element following Kirkland's book ["Advanced Computing in Electron Microscopy"](https://link.springer.com/book/10.1007%2F978-1-4419-6533-2) on an 8x supersampled grid, then integrate it, and store it.

~~~c++
	void fetch_potentials(Array3D<PRISM_FLOAT_PRECISION>& potentials,
	                      const vector<size_t>& atomic_species,
	                      const Array1D<PRISM_FLOAT_PRECISION>& xr,
	                      const Array1D<PRISM_FLOAT_PRECISION>& yr){
		Array2D<PRISM_FLOAT_PRECISION> cur_pot;
		for (auto k =0; k < potentials.get_dimk(); ++k){
			Array2D<PRISM_FLOAT_PRECISION> cur_pot = projPot(atomic_species[k], xr, yr);
			for (auto j = 0; j < potentials.get_dimj(); ++j){
				for (auto i = 0; i < potentials.get_dimi(); ++i){
					potentials.at(k,j,i) = cur_pot.at(j,i);
				}
			}
		}
	}
	
	Array2D<PRISM_FLOAT_PRECISION> projPot(const size_t &Z,
                                           const Array1D<PRISM_FLOAT_PRECISION> &xr,
                                           const Array1D<PRISM_FLOAT_PRECISION> &yr) {
		// compute the projected potential for a given atomic number following Kirkland

		// setup some constants
		static const PRISM_FLOAT_PRECISION pi = std::acos(-1);
		PRISM_FLOAT_PRECISION ss    = 8;
		PRISM_FLOAT_PRECISION a0    = 0.5292;
		PRISM_FLOAT_PRECISION e     = 14.4;
		PRISM_FLOAT_PRECISION term1 = 4*pi*pi*a0*e;
		PRISM_FLOAT_PRECISION term2 = 2*pi*pi*a0*e;

		// initialize array
		ArrayND<2, std::vector<PRISM_FLOAT_PRECISION> > result = zeros_ND<2, PRISM_FLOAT_PRECISION>({{yr.size(), xr.size()}});

		// setup some coordinates
		const PRISM_FLOAT_PRECISION dx = xr[1] - xr[0];
		const PRISM_FLOAT_PRECISION dy = yr[1] - yr[0];

		PRISM_FLOAT_PRECISION start = -(ss-1)/ss/2;
		const PRISM_FLOAT_PRECISION step  = 1/ss;
		const PRISM_FLOAT_PRECISION end   = -start;
		vector<PRISM_FLOAT_PRECISION> sub_data;
		while (start <= end){
			sub_data.push_back(start);
			start+=step;
		}
		ArrayND<1, std::vector<PRISM_FLOAT_PRECISION> > sub(sub_data,{{sub_data.size()}});

		std::pair<Array2D<PRISM_FLOAT_PRECISION>, Array2D<PRISM_FLOAT_PRECISION> > meshx = meshgrid(xr, sub*dx);
		std::pair<Array2D<PRISM_FLOAT_PRECISION>, Array2D<PRISM_FLOAT_PRECISION> > meshy = meshgrid(yr, sub*dy);

		ArrayND<1, std::vector<PRISM_FLOAT_PRECISION> > xv = zeros_ND<1, PRISM_FLOAT_PRECISION>({{meshx.first.size()}});
		ArrayND<1, std::vector<PRISM_FLOAT_PRECISION> > yv = zeros_ND<1, PRISM_FLOAT_PRECISION>({{meshy.first.size()}});
		{
			auto t_x = xv.begin();
			for (auto j = 0; j < meshx.first.get_dimj(); ++j) {
				for (auto i = 0; i < meshx.first.get_dimi(); ++i) {
					*t_x++ = meshx.first.at(j, i) + meshx.second.at(j, i);
				}
			}
		}

		{
			auto t_y = yv.begin();
			for (auto j = 0; j < meshy.first.get_dimj(); ++j) {
				for (auto i = 0; i < meshy.first.get_dimi(); ++i) {
					*t_y++ = meshy.first.at(j, i) + meshy.second.at(j, i);
				}
			}
		}

		std::pair<Array2D<PRISM_FLOAT_PRECISION>, Array2D<PRISM_FLOAT_PRECISION> > meshxy = meshgrid(yv, xv);
		ArrayND<2, std::vector<PRISM_FLOAT_PRECISION> > r2 = zeros_ND<2, PRISM_FLOAT_PRECISION>({{yv.size(), xv.size()}});
		ArrayND<2, std::vector<PRISM_FLOAT_PRECISION> > r  = zeros_ND<2, PRISM_FLOAT_PRECISION>({{yv.size(), xv.size()}});

		{
			auto t_y = r2.begin();
			for (auto j = 0; j < meshxy.first.get_dimj(); ++j) {
				for (auto i = 0; i < meshxy.first.get_dimi(); ++i) {
					*t_y++ = pow(meshxy.first.at(j,i),2) + pow(meshxy.second.at(j,i),2);
				}
			}
		}

		for (auto i = 0; i < r.size(); ++i)r[i] = sqrt(r2[i]);
		// construct potential
		ArrayND<2, std::vector<PRISM_FLOAT_PRECISION> > potSS  = ones_ND<2, PRISM_FLOAT_PRECISION>({{r2.get_dimj(), r2.get_dimi()}});

		// get the relevant table values
		std::vector<PRISM_FLOAT_PRECISION> ap;
		ap.resize(n_parameters);
		for (auto i = 0; i < n_parameters; ++i){
			ap[i] = fparams[(Z-1)*n_parameters + i];
		}

		// compute the potential
		using namespace boost::math;
		std::transform(r.begin(), r.end(),
		               r2.begin(), potSS.begin(), [&ap, &term1, &term2](const PRISM_FLOAT_PRECISION& r_t, const PRISM_FLOAT_PRECISION& r2_t){

					return term1*(ap[0] *
					              cyl_bessel_k(0,2*pi*sqrt(ap[1])*r_t)          +
					              ap[2]*cyl_bessel_k(0,2*pi*sqrt(ap[3])*r_t)    +
					              ap[4]*cyl_bessel_k(0,2*pi*sqrt(ap[5])*r_t))   +
					       term2*(ap[6]/ap[7]*exp(-pow(pi,2)/ap[7]*r2_t) +
					              ap[8]/ap[9]*exp(-pow(pi,2)/ap[9]*r2_t)        +
					              ap[10]/ap[11]*exp(-pow(pi,2)/ap[11]*r2_t));
				});

		// integrate
		ArrayND<2, std::vector<PRISM_FLOAT_PRECISION> > pot = zeros_ND<2, PRISM_FLOAT_PRECISION>({{yr.size(), xr.size()}});
		for (auto sy = 0; sy < ss; ++sy){
			for (auto sx = 0; sx < ss; ++sx) {
				for (auto j = 0; j < pot.get_dimj(); ++j) {
					for (auto i = 0; i < pot.get_dimi(); ++i) {
						pot.at(j, i) += potSS.at(j*ss + sy, i*ss + sx);
					}
				}
			}
		}
		pot/=(ss*ss);

		PRISM_FLOAT_PRECISION potMin = get_potMin(pot,xr,yr);
		pot -= potMin;
		transform(pot.begin(),pot.end(),pot.begin(),[](PRISM_FLOAT_PRECISION& a){return a<0?0:a;});

		return pot;
	}	
~~~

#### Building the sliced potentials

First there's a bit of setup and we figure out which slice index each atom belongs to

~~~ c++
	void generateProjectedPotentials(Parameters<PRISM_FLOAT_PRECISION>& pars,
	                                 const Array3D<PRISM_FLOAT_PRECISION>& potentialLookup,
	                                 const vector<size_t>& unique_species,
	                                 const Array1D<long>& xvec,
	                                 const Array1D<long>& yvec){
		// splits the atomic coordinates into slices and computes the projected potential for each.

		// create arrays for the coordinates
		Array1D<PRISM_FLOAT_PRECISION> x     = zeros_ND<1, PRISM_FLOAT_PRECISION>({{pars.atoms.size()}});
		Array1D<PRISM_FLOAT_PRECISION> y     = zeros_ND<1, PRISM_FLOAT_PRECISION>({{pars.atoms.size()}});
		Array1D<PRISM_FLOAT_PRECISION> z     = zeros_ND<1, PRISM_FLOAT_PRECISION>({{pars.atoms.size()}});
		Array1D<PRISM_FLOAT_PRECISION> ID    = zeros_ND<1, PRISM_FLOAT_PRECISION>({{pars.atoms.size()}});
		Array1D<PRISM_FLOAT_PRECISION> sigma = zeros_ND<1, PRISM_FLOAT_PRECISION>({{pars.atoms.size()}});

		// populate arrays from the atoms structure
		for (auto i = 0; i < pars.atoms.size(); ++i){
			x[i]     = pars.atoms[i].x * pars.tiledCellDim[2];
			y[i]     = pars.atoms[i].y * pars.tiledCellDim[1];
			z[i]     = pars.atoms[i].z * pars.tiledCellDim[0];
			ID[i]    = pars.atoms[i].species;
			sigma[i] = pars.atoms[i].sigma;
		}

		// compute the z-slice index for each atom
		auto max_z = std::max_element(z.begin(), z.end());
		Array1D<PRISM_FLOAT_PRECISION> zPlane(z);
		std::transform(zPlane.begin(), zPlane.end(), zPlane.begin(), [&max_z, &pars](PRISM_FLOAT_PRECISION &t_z) {
			return round((-t_z + *max_z) / pars.meta.sliceThickness + 0.5) - 1; // If the +0.5 was to make the first slice z=1 not 0, can drop the +0.5 and -1
		});
		max_z = std::max_element(zPlane.begin(), zPlane.end());
		pars.numPlanes = *max_z + 1;
	
#ifdef PRISM_BUILDING_GUI
		pars.progressbar->signalPotentialUpdate(0, pars.numPlanes);
#endif

~~~

A new idiom you see here is `#ifdef PRISM_BUILDING_GUI`. There is a progress bar that pops up in the GUI, and this macro is used to indicate whether or not to update it. When this source file is included in the CLI, `prism`, this macro is not defined, and thus the CLI doesn't need any of the Qt libraries that would be required to define this progress bar.

Next we calculate the actual potentials, and this is the first time we encounter the `WorkDispatcher` in action. In this case, each slice of the potential is a different job, and  CPU threads will be spawned to populate each slice.

~~~ c++
		// initialize the potential array
		pars.pot = zeros_ND<3, PRISM_FLOAT_PRECISION>({{pars.numPlanes, pars.imageSize[0], pars.imageSize[1]}});
~~~

As an aside, this `zeros_ND` function is essentially an implementation of MATLAB's `zeros` and creates a multidimensional array built on `std::vector` and is 0-initialized. 

Now we make the `WorkDispatcher`, passing the lower and upper bounds of the work IDs into its constructor. The threads are then spawned and handed a lambda function defining the work loop. Each thread repeatedly calls `dispatcher.getWork`, which you'll recall from earlier returns true if a job was given. As long as work was provided, the thread will keep working, and once all the work is done the `WorkDispatcher` will begin returning false and the threads will finish. The main thread doesn't do any work -- it spawns the worker threads and then will wait at the line  `for (auto &t:workers)t.join();` until all the worker threads finish. The `join()` function is how you synchronize this multithreaded code. So the only parts that require synchronization are when each threads queries the `WorkDispatcher` and then when the main thread waits for the workers. The order in which the slices are worked on is nondeterministic. By the way, if you are wondering the cost of locking a mutex to hand out a work ID to each thread is completely negligible compared to how long it takes to do the work. The `WorkDispatcher` synchronization is effectively free.

~~~ c++
		// create a key-value map to match the atomic Z numbers with their place in the potential lookup table
		map<size_t, size_t> Z_lookup;
		for (auto i = 0; i < unique_species.size(); ++i)Z_lookup[unique_species[i]] = i;

		//loop over each plane, perturb the atomic positions, and place the corresponding potential at each location
		// using parallel calculation of each individual slice
		std::vector<std::thread> workers;
		workers.reserve(pars.meta.NUM_THREADS);

		WorkDispatcher dispatcher(0, pars.numPlanes);
		for (long t = 0; t < pars.meta.NUM_THREADS; ++t){
			cout << "Launching thread #" << t << " to compute projected potential slices\n";
			workers.push_back(thread([&pars, &x, &y, &z, &ID, &Z_lookup, &xvec, &sigma,
											 &zPlane, &yvec,&potentialLookup, &dispatcher](){
				// create a random number generator to simulate thermal effects
				std::default_random_engine de(pars.meta.random_seed);
				normal_distribution<PRISM_FLOAT_PRECISION> randn(0,1);
				Array1D<long> xp;
				Array1D<long> yp;

				size_t currentBeam, stop;
                currentBeam=stop=0;
				while (dispatcher.getWork(currentBeam, stop)) { // synchronously get work assignment
					Array2D<PRISM_FLOAT_PRECISION> projectedPotential = zeros_ND<2, PRISM_FLOAT_PRECISION>({{pars.imageSize[0], pars.imageSize[1]}});
					while (currentBeam != stop) {
						for (auto a2 = 0; a2 < x.size(); ++a2) {
							if (zPlane[a2] == currentBeam) {
								const long dim0 = (long) pars.imageSize[0];
								const long dim1 = (long) pars.imageSize[1];
								const size_t cur_Z = Z_lookup[ID[a2]];
								PRISM_FLOAT_PRECISION X, Y;
								if (pars.meta.include_thermal_effects) { // apply random perturbations
									X = round((x[a2] + randn(de) * sigma[a2]) / pars.pixelSize[1]);
									Y = round((y[a2] + randn(de) * sigma[a2]) / pars.pixelSize[0]);
								} else {
									X = round((x[a2]) / pars.pixelSize[1]); // this line uses no thermal factor
									Y = round((y[a2]) / pars.pixelSize[0]); // this line uses no thermal factor
								}
								xp = xvec + (long) X;
								for (auto &i:xp)i = (i % dim1 + dim1) % dim1; // make sure to get a positive value

								yp = yvec + (long) Y;
								for (auto &i:yp) i = (i % dim0 + dim0) % dim0;// make sure to get a positive value
								for (auto ii = 0; ii < xp.size(); ++ii) {
									for (auto jj = 0; jj < yp.size(); ++jj) {
										// fill in value with lookup table
										projectedPotential.at(yp[jj], xp[ii]) += potentialLookup.at(cur_Z, jj, ii);
									}
								}
							}
						}
						// copy the result to the full array
						copy(projectedPotential.begin(), projectedPotential.end(),&pars.pot.at(currentBeam,0,0));
#ifdef PRISM_BUILDING_GUI
                        pars.progressbar->signalPotentialUpdate(currentBeam, pars.numPlanes);
#endif //PRISM_BUILDING_GUI
						++currentBeam;
					}
				}
			}));
		}
		cout << "Waiting for threads...\n";
		for (auto &t:workers)t.join();
#ifdef PRISM_BUILDING_GUI
		pars.progressbar->setProgress(100);
#endif //PRISM_BUILDING_GUI
	};
~~~