# Prismatic Tutorial (Under Construction)   
---
*This tutorial is a work in progress and will be finished in the very near future*

## Table of Contents  
- [Examples](#examples)
- [Prismatic GUI simulation of algorithm paper decahedral nanoparticle](#tutorialdeca)
- [PyPrismatic: Using Prismatic through Python](#pyprismatic)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  - [Metadata Parameters](#metadata)



## Examples

Within the source code of `Prismatic` is an "examples" folder that contains some example scripts of how one might run simulations using *Prismatic*. There is a mixture of Python examples use `PyPrismatic` and bash shell scripts using `prismatic`. Both tools are accessing the exact same underlying code -- they just provide different entry points. Use whatever you feel most comfortable with.



<a name="tutorialdeca"></a>
## Prismatic GUI simulation of algorithm paper decahedral nanoparticle

Placeholder



<a name="pyprismatic"></a>
## PyPrismatic: Using Prismatic through Python  

*Instructions for installing `PyPrismatic` may be found [here](www.prism-em.com/installation/)*

To run a simulation with `PyPrismatic`, you simple create a `Metadata` object, adjust the parameters, and then execute the calculation with the `go` method. A list of adjustable parameters is [below](#metadata). These parameters can either be set with keyword arguments when constructing the `Metadata` object, or directly with the `.` operator. A simple example script utilizing both methods of setting parameters follows where the hypothetical input atomic coordinate information exists in the file "myInput.XYZ", the electron energy is set to 100 keV, and a 3x3x3 unit cell tiling is desired. The remaining parameters will be set to the default values (the `toString()` method can be used to print out all of the current settings).

~~~ python
import pyprismatic as pr
meta = pr.Metadata(filenameAtoms="myInput.XYZ", E0=100e3)
meta.tileX = meta.tileY = meta.tileZ = 3
meta.go()
~~~



<a name="metadata"></a>
## List of Metadata parameters

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
alphaBeamMax** : the maximum probe angle to consider (in mrad)  
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
**probeXtilt** : (in Angstroms)  
**probeYtilt** : (in Angstroms)  
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
**integrationAngleMin** : (in mrad)  
**integrationAngleMax** : (in mrad)  
**transferMode** : memory model to use, either "streaming", "singlexfer", or "auto"  
