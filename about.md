# About *Prismatic*
---

Table of Contents    
  - [File Formats](#file-formats)  
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - [Input](#input)  
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - [Output](#output)  
  - [GPU Compute Capability](#gpu-compute-capability)
  - [Command Line Options](#cli-options)
  - [List of `PyPrismatic` Metadata parameters](#pyprismatic-metadata)

*Prismatic* is a CUDA/C++/Python software package for fast image simulation in scanning transmission electron microscopy (STEM). It includes parallel, data-streaming implementations of both the plane-wave reciprocal-space interpolated scattering matrix (PRISM) and multislice algorithms using multicore CPUs and CUDA-enabled GPU(s), in some cases achieving accelerations as high as 1000x or more relative to traditional methods. *Prismatic* is fast, free, open-sourced, and contains a graphical user interface.


## File Formats

### Input

Input files containing the atomic coordinates should be in the following XYZ format.

* 1 comment line
* 1 line with the 3 unit cell dimensions a,b,c (in Angstroms)
* *N* lines of the comma-separated form `Z`, `x`, `y`, `z`, `occ`, `thermal_sigma` where `Z` is the atomic number, `x`, `y`, and `z` are the coordinates of the atom within the unit cell in Angstroms, `occ` is the occupancy number (value between 0 and 1 that specifies the likelihood that an atom exists at that site), and `thermal_sigma` is the standard deviation of random thermal motion in Angstroms (Debye-Waller effect).
* -1 to indicate end of file

Here is a sample file, "SI100.XYZ" taken from `computem`, which uses the same format

~~~
one unit cell of (100) silicon
      5.43    5.43    5.43
  14  0.0000  0.0000  0.0000  1.0  0.076
  14  2.7150  2.7150  0.0000  1.0  0.076
  14  1.3575  4.0725  1.3575  1.0  0.076
  14  4.0725  1.3575  1.3575  1.0  0.076
  14  2.7150  0.0000  2.7150  1.0  0.076
  14  0.0000  2.7150  2.7150  1.0  0.076
  14  1.3575  1.3575  4.0725  1.0  0.076
  14  4.0725  4.0725  4.0725  1.0  0.076
  -1
~~~

### Output

Outputs are written to binary [.mrc](http://bio3d.colorado.edu/imod/doc/mrc_format.txt) files with float-precision. There are potentially 2D, 3D, and 4D output. For all outputs, there are at least two dimensions corresponding to X and Y probe positions. At each position, `Prismatic` can output the full probe (4D), a radially integrated ouput into virtual detector bins (3D), or further integrated over a range of detector bins to produce a single value for each scan position (2D). The 3D output is considered to be the primary result and is the only output produced by default; however, any combination of 2D, 3D, and 4D outputs may be produced with a single simulation. The metadata parameter `filename_output`, set with "-o" at the command line, is used as a base filename and modified depending on the output type.

* 2D output: Produced by adding the command line option "-2D ang\_min ang\_max" where "ang\_min" and "ang\_max" are the inner and outer integration angles in mrad. The resulting image will be composed as either "prism\_2Doutput\_" + `filename_output` or "multislice\_2Doutput\_" + `filename_output`. By default this is off.
* 3D output: Controlled by command line option "-3D 0/1" where 0 or 1 is a boolean on/off. The 3D output is saved to `filename_output`. By default this is on.
* 4D output: Controlled by command line option "-4D 0/1" where 0 or 1 is a boolean on/off. The 4D output saves a separate 2D MRC image for each X/Y probe scan position. The output name for each image is tagged with the X and Y scan position index + `filename_output`. By default this is off.

For example, the `prismatic` command 
`./prismatic -i atoms.XYZ -2D 0 10 -4D 1 -3D 1 -o example.mrc`
will produce "prism\_2Doutput\_example.mrc" with the 2D bright field image integrated from 0-10 mrad, the 3D output in "example.mrc", and the 4D output consisting of many individual 2D images with names of the form  "example\_X##\_Y##\_FP##.mrc" where the number values indicate the integer index of the scan in X and Y, accordingly. For example, if the simulation parameters are such that the probe step size is 1 Angstrom, then the file "example\_X1_Y2_FP2.mrc" contains the 2D intensity values corresponding to probe position (1.0, 2.0) Angstroms for the second frozen phonon configuration. For the time being, it will likely require some scripting on the user's part to wrangle the 4D output. In the future, we intend to introduce an hdf5 format to contain each of these outputs in a unified way. 

## GPU Compute Capability

The GPU version of `Prismatic` requires a CUDA-enabled GPU with compute capability >= 3.0

<a name ="cli-options"></a>
## Using `Prismatic` from the command line

`Prismatic` contains a command line tool, `prismatic`, that can be used to run simulations from within a terminal, bash script, etc. Building it requires the `CMake` variable `PRISM_ENABLE_CLI=1` at compilation time, which is the default behavior.

The following options are available with `prismatic` (you can also print available options and default values with `prismatic --help`), each documented as **_long form_** **_(short form)_** *parameters* : description

* --input-file (-i) filename : filename containing input atom information in XYZ format (see [here](http://prism-em.com/about/) for more details)  
* --output-file(-o) filename : base output filename
* --interp-factor (-f) number : PRISM interpolation factor, used for both X and Y
* --interp-factor-x (-fx) number : PRISM interpolation factor in X
* --interp-factor-y (-fy) number : PRISM interpolation factor in Y
* --num-threads (-j) value : number of CPU threads to use
* --num-streams (-S) value : number of CUDA streams to create per GPU
* --num-gpus (-g) value : number of GPUs to use. A runtime check is performed to check how many are actually available, and the minimum of these two numbers is used.
* --batch-size (-b) value : number of probes/beams to propagate simultaneously for both CPU and GPU workers.
* --batch-size-cpu (-bc) value : number of probes/beams to propagate simultaneously for CPU workers.
* --batch-size-gpu (-bg) value : number of probes/beams to propagate simultaneously for GPU workers.
* --slice-thickness (-s) thickness : thickness of each slice of projected potential (in Angstroms)
* --help(-h) : print information about the available options
* --pixel-size (-p) pixel_size : size of simulation pixel size
* --detector-angle-step (-d) step_size : angular step size for detector integration bins (in mrad)
* --cell-dimension (-c) x y z : size of sample in x, y, z directions (in Angstroms)
* --tile-uc (-t) x y z : tile the unit cell x, y, z number of times in x, y, z directions, respectively
* --algorithm (-a) p/m : the simulation algorithm to use, either (p)rism or (m)ultislice
* --energy (-E) value : the energy of the electron beam (in keV)
* --alpha-max (-A) angle : the maximum probe angle to consider (in mrad)
* --potential-bound (-P) value : the maximum radius from the center of each atom to compute the potental (in Angstroms)
* --also-do-cpu-work (-C) bool=true : boolean value used to determine whether or not to also create CPU workers in addition to GPU ones
* --streaming-mode 0/1 : boolean value to force code to use (true) or not use (false) streaming versions of GPU codes. The default behavior is to estimate the needed memory from input parameters and choose automatically.
* --probe-step (-r) step_size : step size of the probe for both X and Y directions (in Angstroms)
* --probe-step-x (-rx) step_size : step size of the probe in X direction (in Angstroms)
* --probe-step-y (-ry) step_size : step size of the probe in Y direction (in Angstroms)
* --random-seed (-rs) step_size : random number seed, integer
* --probe-xtilt (-tx) value : probe X tilt (in mrad)
* --probe-ytilt (-ty) value : probe X tilt (in mrad)
* --probe-defocus (-df) value : probe defocus (in Angstroms)
* --probe-semiangle (-sa) value : maximum probe semiangle (in mrad)
* --scan-window-x (-wx) min max : size of the window to scan the probe in X (in fractional coordinates between 0 and 1)
* --scan-window-y (-wy) min max : size of the window to scan the probe in Y (in fractional coordinates between 0 and 1)
* --num-FP (-F) value : number of frozen phonon configurations to calculate
* --thermal-effects (-te) bool : whether or not to include Debye-Waller factors (thermal effects)
* --occupancy (-oc) bool : whether or not to consider occupancy values for likelihood of atoms existing at each site
* --save-2D-output (-2D) ang_min ang_max : save the 2D STEM image integrated between ang_min and ang_max (in mrads)
* --save-3D-output (-3D) bool=true : Also save the 3D output at the detector for each probe (3D output mode)
* --save-4D-output (-4D) bool=false : Also save the 4D output at the detector for each probe (4D output mode)



<a name="pyprismatic-metadata"></a>
## List of `PyPrismatic` Metadata parameters

**interpolationFactorX** : PRISM interpolation factor in x-direction  
**interpolationFactorY** : PRISM interpolation factor in y-direction  
**filenameAtoms** : filename containing input atom information in XYZ format (see [here](http://prism-em.com/about/) for more details)  
**filenameOutput** : filename in which to save the 3D output. Also serves as base filename for 2D and 4D outputs if used  
**realspacePixelSizeX** : size of pixel size in X for probe/potential arrays  
**realspacePixelSizeY** : size of pixel size in Y for probe/potential arrays  
**potBound** : limiting radius within which to compute projected potentials from the center of each atom (in Angstroms)  
**numFP** : number of frozen phonon configurations to average over  
**sliceThickness** : thickness of potential slices (in Angstroms)  
**cellDimX** : unit cell dimension X (in Angstroms)  
**cellDimY** : unit cell dimension Y (in Angstroms)  
**cellDimZ** : unit cell dimension Z (in Angstroms)  
**tileX** : number of unit cells to tile in X direction  
**tileY** : number of unit cells to tile in Y direction  
**tileZ** : number of unit cells to tile in Z direction  
**E0** : electron beam energy (in eV)  
**alphaBeamMax** : the maximum probe angle to consider (in mrad)  
**numGPUs** : number of GPUs to use. A runtime check is performed to check how many are actually available, and the minimum of these two numbers is used   
**numStreamsPerGPU** : number of CUDA streams to use per GPU  
**numThreads** : number of CPU worker threads to use  
**batchSizeTargetCPU** : desired batch size for CPU FFTs  
**batchSizeTargetGPU** : desired batch size for GPU FFTs  
**earlyCPUStopCount** : the WorkDispatcher will cease providing work to CPU workers earlyCPUStopCount jobs from the end. This is to prevent the program waiting for slower CPU workers to complete  
**probeStepX** : step size of the probe in X direction (in Angstroms)  
**probeStepY** : step size of the probe in Y direction (in Angstroms)  
**probeDefocus** : probe defocus (in Angstroms)  
**C3** : microscope C3 (in Angstroms)    
**probeSemiangle** : probe convergence semi-angle (in mrad)  
**detectorAngleStep** : angular step size for detector integration bins (in mrad)  
**probeXtilt** : probe X tilt (in mrad)  
**probeYtilt** : probe X tilt (in mrad)  
**scanWindowXMin** : lower X size of the window to scan the probe (in fractional coordinates)  
**scanWindowXMax** : upper X size of the window to scan the probe (in fractional coordinates)  
**scanWindowYMin** : lower Y size of the window to scan the probe (in fractional coordinates)  
**scanWindowYMax** : upper Y size of the window to scan the probe (in fractional coordinates)  
**randomSeed** : number to use for random seeding of thermal effects  
**algorithm** : simulation algorithm to use, "prism" or "multislice"  
**includeThermalEffects** : true/false to apply random thermal displacements (Debye-Waller effect)  
**alsoDoCPUWork** : true/false to spawn CPU workers in addition to GPU workers  
**save2DOutput** : save the 2D STEM image integrated between integrationAngleMin and integrationAngleMax  
**save3DOutput** : true/false Also save the 3D output at the detector for each probe (3D output mode)  
**save4DOutput** : true/false Also save the 4D output at the detector for each probe (4D output mode)  
**integrationAngleMin** : inner detector position (for 2D output mode) (in mrad)  
**integrationAngleMax** : outer detector position (for 2D output mode) (in mrad)  
**transferMode** : memory model to use, either "streaming", "singlexfer", or "auto"  
