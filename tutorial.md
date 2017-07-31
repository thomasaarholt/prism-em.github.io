# Prismatic Tutorial (Under Construction)   
---
*This tutorial is a work in progress and will be finished in the very near future*

## Table of Contents  
- [Examples](#examples)
- [Prismatic GUI simulation of decahedral NP](#tutorialdeca)
- [PyPrismatic: Using Prismatic through Python](#pyprismatic)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  - [Metadata Parameters](#metadata)



## Examples

Within the source code of `Prismatic` is an "examples" folder that contains some example scripts of how one might run simulations using *Prismatic*. There is a mixture of Python examples use `PyPrismatic` and bash shell scripts using `prismatic`. Both tools are accessing the exact same underlying code -- they just provide different entry points. Use whatever you feel most comfortable with.





<a name="tutorialdeca"></a>
## Prismatic GUI simulation of decahedral NP

[Issues with this tutorial?  Email Colin at clophus@lbl.gov]

In this example we will walk you through the entire procedure for generating a STEM simulation of a decahedral nanoparticle. This simulation should match the result shown in the PRISM algorithm paper: [DOI:10.1186/s40679-017-0046-1](http://dx.doi.org/10.1186/s40679-017-0046-1), specifically Figures 3, 4 and 5. Below is an overview of the sample we will construct, and an example of image simulations generated from both the multislice method, and the PRISM algorithm with varying degrees of accuracy.



| ![animated decahedral nanoparticle](img/decaRotate01.gif) | ![example STEM image simulations](img/deca_sim_01_overview.png) | 
|:---:|:---:|
| Atomic model of a decahedral nanoparticle resting upon an amorphous carbon substrate. | STEM annular bright field and annular dark field image simulations taken from Figure 4 of the [PRISM algorithm paper](http://dx.doi.org/10.1186/s40679-017-0046-1). |


The steps we will follow for this tutorial are:

1. Download atomic coordinate files.
2. Construct unified atomic model in `Matlab`.
3. Export coordinates in .xyz format for `Prismatic`.
4. Import atoms into Prismatic and set parameters.
5. Test PRISM vs Multislice accuracy.
6. Run PRISM simulation, save output as .mrc.
7. Generate final image outputs.

### 1 - Download atomic coordinate files.

To simulate a realistic nanoparticle sample, we require two sets of atomic coordinates. The first is (obviously) the nanoparticle itself. We have chosen to simulate a defected nanoparticle, as this is a more realistic sample with more interesting features than the ideal nanoparticles often used in atomic model simulations. Such a structure is available for download from a page on the [Miao group website](http://www.physics.ucla.edu/research/imaging/dislocations/index.html). I've saved the coordinates as a .csv file [here](data/atoms_deca_cartesian_x_y_z_ID.csv). Each row corresponds to an atom, and the four columns are [x y z ID] where (x,y,z) are the 3D position in Angstroms, and ID is the atomic Z number of the atoms. In this case I've used 78 for platinum, but this can be changed to any atomic species for this example simulation.

This nanoparticle has a defected decahedral geometry; a type of multiply-twinned particle composed of 5 tetrahedral crystalline FCC segments (see [Wikipedia/Pentagonal_bipyramid](https://en.wikipedia.org/wiki/Pentagonal_bipyramid) for an stretched image of the shape). The 5 boundaries between these 5 segments are FCC crystalline twins. In this decahedral particle, 2 of the 5 segments contain defects - stacking faults and dislocations.  This makes the particle interesting from a imaging standpoint, since the symmetry breaking of the defects leads to complex diffraction signals even when viewing down crystallographic axes.

The second set of coordinates is often overlooked in image simulations - we need a block of atoms representing a realistic substrate.  A substrate is an unavoidable part of electron microscopy experiments, since samples stubbornly refuse to float in free space in the path of the electron beam. In order to affect the experiment as little as possible, we typically utilize very thin, low atomic number substrates, usually ultra-thin amorphous carbon supports. We could just randomly generate coordinates and then delete the atoms that are too close together, but the literature contains better examples. Let's use the amorphous carbon blocks given in [this paper](http://dx.doi.org/10.1063/1.4831669) by Ricolleau et al. These authors have (generously) made some of their realistic amorphous carbon structures available for download. In case you don't have access I have again saved the coordinates as a .csv file [here](data/atoms_amorCarbon_50nmCube_x_y_z_ID.csv),

We now have all of the atomic coordinates required to build our simulation "scene."




### 2 - Construct unified atomic model in `Matlab`.

This section can be completed using any interactive programming language. I will describe the procedure using `Matlab` code as an example, which can be easily adapted to the language of your choice. First, import the data into `Matlab` (right click on the .csv files and select "Import Data," naming the two arrays something suitable).  Next, we perform a few simple steps:

* Tile 5x5x5nm amorphous carbon cube 4 times to form a 10x10x5 nm substrate.
* Tilt nanoparticle to desired orientation.
* Position nanoparticle in center of cell, just above / slightly inside the substrate.
* Delete substrate atoms too close to / overlapping nanoparticle atoms.

First, let's tile the subtrace block 2x2 times to make it large enough to hold the nanoparticle, without periodic wrap-around artifacts.  We will also permute the dimensions of 3 out of 4 blocks to prevent tiling artifacts.  The boundaries will not be perfectly physical, but this error will be small.  Assuming we import the xyz coordinates of the substrate block as "xyzSub" we can use the below code to tilt the subtrate (and cell boundaries) by 2x2x1:

```matlab
cellSub = [50 50 50];       % Original substrate cell size
cellDim = [2*cellSub(1:2) cellSub(3)];
atomsSub = [ ...
    xyzSub(:,[1 2 3])+repmat([0          0          0],[size(xyzSub,1) 1]);
    xyzSub(:,[2 1 3])+repmat([cellSub(1) 0          0],[size(xyzSub,1) 1]);
    xyzSub(:,[3 1 2])+repmat([0          cellSub(2) 0],[size(xyzSub,1) 1]);
    xyzSub(:,[2 1 3])+repmat([cellSub(1:2)          0],[size(xyzSub,1) 1])];
```

Next, I've chosen to rotate the nanoparticle by 30 degrees around the x-axis, placing the 2 defected sectors onto a "low index" zone axis (atomic columns far apart) and the other 3 sectors onto a "high index" zone axis (atomic columns closer together).  I typically use "z-x-z" rotation for my three [Euler angles](https://en.wikipedia.org/wiki/Euler_angles), for example in the code below: (note I also translate the particle to sit "on top" of the now 10x10x5 nm substrate block)

```matlab
theta = [0 -30 0]*pi/180; % 3 angles for ZXZ rotation of NP
shiftNP = [50 50 60.2];   % NP (x,y,z) position
xyzNP = xyzNP(:,1:3);     % Remove columns >3 in the NP array
for a0 = 1:3  % Center on (0,0,0)
    xyzNP(:,a0) = xyzNP(:,a0) - mean(xyzNP(:,a0));  
end
% Rotate and translate NP
m = [cos(theta(1)) -sin(theta(1)) 0;
    sin(theta(1)) cos(theta(1)) 0;
    0 0 1];
xyzNP = xyzNP * m;
m = [1 0 0;
    0 cos(theta(2)) -sin(theta(2));
    0 sin(theta(2)) cos(theta(2))];
xyzNP = xyzNP * m;
m = [cos(theta(3)) -sin(theta(3)) 0;
    sin(theta(3)) cos(theta(3)) 0;
    0 0 1];
xyzNP = xyzNP * m;
for a0 = 1:3
    xyzNP(:,a0) = xyzNP(:,a0) + shiftNP(a0);
end
```

Now we need to delete any overlapping atoms between the substrate and NP.  This is easily done by testing each substrate atom to see if it is too close to the atom inside the nanoparticle that it is closest too. Below I've used a cutoff separation of 3 Angstroms.

```matlab
rMin = 3;                   % minimum atomic separation
r2 = rMin^2;
del = false(size(atomsSub,1),1);
for a0 = 1:size(atomsSub,1)
    if (  min((atomsSub(a0,1)-xyzNP(:,1)).^2 ...
            + (atomsSub(a0,2)-xyzNP(:,2)).^2 ...
            + (atomsSub(a0,3)-xyzNP(:,3)).^2) < r2)
        del(a0) = true;
    end
end
atomsSub(del,:) = [];
```

You can find the above code (with some included plotting code) at [this link](data/build_scene_deca_tilted_on_amorphous_carbon.m).  The visualizations generated by this `Matlab` script are included below for reference. This code also adds a 4th atomic column, where you can specify the species of both the nanoparticle and substrate - which I have set to platinum and carbon respectively.  Note that in Prismatic, the beam direction is always along the third (z) direction of the cell.  We also do not (currently) allow non-orthogonal cell boundaries, which may require you to tile your unit cell to form an orthorhombic or pseudo-orthorhombic cell before running the simulation.



| ![animated decahedral nanoparticle](img/deca_amorphous_carbon_scatter_plot.png) | ![example STEM image simulations](img/deca_amorphous_carbon_RGB_image3.jpg) | 
|:---:|:---:|
| Side view scatter plot of the atomic coordinates, Pt nanoparticle in red and carbon substrate in green. | Top down view of nanoparticle plus substrate, rendered as an RGB image. |





### 3 - Export coordinates in .xyz format for `Prismatic`.

`Prismatic` uses the same .xyz input format as `computem`.  As in most .xyz files, the first two rows are reserved for comments and each following row contains "space-separated" values. The first row can be set to anything, typically it is used for a descriptive title of the simulation cell. We will use the second row to list the three values specified above as "cellDim" which represent the size of the simulatin cell in Angstroms.  All following rows consist of columns containing six values:

1. atomic number
2. x 
3. y 
4. z 
5. occupancy 
6. RMS thermal vibration

where the coordinates (x,y,z) are cartesian values in Angstroms, the occupancy is a number from 0 to 1 representing how frequently that site contains an atom, and the final value is the root-mean-square (RMS) thermal vibration coefficient, also in Angstroms.  The vibration of most atoms at room temperature follows a Gaussian distribution with a standard deviation of 0.05-0.1 Angstroms.  So, for the simulation file described above, the first several lines will look like:

```csv
Atomic coordinates for decahedral nanoparticle on amorphous carbon
   100.00000  100.00000  80.000000
6  30.728000  40.226000  3.720700  1  0.080000
6  28.026000  39.412000  6.253700  1  0.080000
6  21.581000  47.227000  3.620000  1  0.080000
6  22.244000  44.872000  3.643700  1  0.080000
...
```
 
Lastly, to maintain compatibility with `computem` we typically write "-1" in the last or second last line of the .xyz file. A simple `Matlab` script that writes .xyz files in the above format  is given here:

```matlab
function [] = writeXYZ(fileName,comment,cellDim,IDarray,xyzArray,occArray,uArray)
% Write .xyz file for Prismatic

if length(IDarray) == 1
    IDarray = IDarray*ones(size(xyzArray,1),1);
end
if length(occArray) == 1
    occArray = occArray*ones(size(xyzArray,1),1);
end
if length(uArray) == 1
    uArray = uArray*ones(size(xyzArray,1),1);
end

% Initialize file
fid = fopen(fileName,'w');
% Write comment line (1st)
fprintf(fid,[comment '\n']);
% Write cell dimensions
fprintf(fid,'    %f %f %f\n',cellDim(1:3));
% Write atomic data
dataAll = [IDarray xyzArray occArray uArray];
for a0 = 1:size(dataAll,1)
    fprintf(fid,'%d  %f  %f  %f  %d  %f\n',dataAll(a0,:));
end

% Write end of file, for computem compatibility
fprintf(fid,'-1\n');
% Close file
fclose(fid);

end
```
This script can be downloaded [here](data/writeXYZ.m), and the final .xyz output for the above cell containing a rotated decahedral NP on an amorphous carbon substrate can be downloaded at [here](data/AuDeca_amorCarbon.xyz). This `Matlab` script requires string inputs for "fileName" and "comment," the three value vector cellDim, the array of atomic IDs and xyz coordinates (in Angstroms), and finally arrays of all occupancy and RMS displacement values in occArray amd uArray respectively. Converting fractional atomic coordinates to cartesian coordinates in Angstroms can be done inline:

```matlab
writeXYZ('AuDeca_amorCarbon.xyz',...
'Atomic coordinates for decahedral nanoparticle on amorphous carbon',...
cellDim,atoms(:,4),atoms(:,1:3).*repmat(cellDim,[size(atoms,1) 1]),1,0.08);
```



### 4 - Import atoms into Prismatic and set parameters.
[In Progress]

### 5 - Test PRISM vs Multislice accuracy.
[In Progress]

### 6 - Run PRISM simulation, save output as .mrc.
[In Progress]

### 7 - Generate final image outputs.
[In Progress]



<a name="pyprismatic"></a>
## PyPrismatic: Using Prismatic through Python  

*Instructions for installing `PyPrismatic` may be found at [here](www.prism-em.com/installation/)*

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
