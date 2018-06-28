

![Prismatic screenshot 01](img/SuperSTEMtopbar.png){:width="1184px"}


## Table of Contents  
- [1 - Install `Prismatic`](#step1)
- [2 - Load atomic coordinates](#step2)
- [3 - Set microscope parameters](#step3)
- [4 - Projected atomic potentials](#step4)
- [5 - Unit cell tiling and PRISM accuracy](#step5)
- [6 - Run simulations](#step6)
- [7 - Save results](#step7)
- [8 - Further simulations](#step8)




&nbsp;
<a name="step1"></a>
## 1 - Download and Install the GUI Version of Prismatic

The `Prismatic` GUI has been compiled for both Windows and OSX. For each of these operating systems, there are two possible versions of Prismatic you may wish to install.

### For Windows:

If your computer has an NVIDIA GPU that can run CUDA code, install:
[Prismatic-GPU-v1.1.msi](https://drive.google.com/open?id=1B9Yq-BBWY3VvNRD-aiKTWGzDME7s8qPh)

Otherwise, install:
[Prismatic-v1.1.msi](https://drive.google.com/open?id=13TZZc1ZAzMMx-cmfiCJCL4pBPqW_icWS)

### For OSX:

If you are running OSX 10.13.1 (High Sierra), install:
[Prismatic-OSX-v1.1.dmg (High Sierra)](https://drive.google.com/open?id=1OnclVmfDv9oIAXVdTbq6dp94DDiLuWfk)

If you are running OSX 10.12.6 (Sierra), install:
[Prismatic-OSX-v1.1.dmg (Sierra)](https://drive.google.com/open?id=1S1utdTErovvkf-o5P4gTRB5IeC4smYqZ)

The GPU portions of `Prismatic` were developed using CUDA. This means that Apple laptops and Mac desktops running AMD GPUs cannot make use of CUDA GPU code. However, the PRISM algorithm is quite fast. Therefore you may be able to run STEM simulations on Apple laptops in reasonable times.

After downloading and installing `Prismatic`, run it to verify the installation has succeeded.



&nbsp;
<a name="step2s"></a>
## 2 - Download atomic coordinate files, load into Prismatic


The sample we are going to examine in this tutorial is barium neodymium titanate. This material has a complex unit cell, with the tungsten bronze parent structure. From this unit cell, we have constructed 5 different zone axes by projecting the unit cell into new pseudo-orthogonal unit cells. This methodology is described below.

`Prismatic` uses the same .xyz file format as Kirkland's computem software. We also assume the beam propagation direction is along the z axis. This file format contains 2 header lines, a comment and the cell dimensions (along x, y and z respectively). Next, the atom coordinates are listed in rows with these 6 column values:

1. atomic number
2. x 
3. y 
4. z 
5. occupancy 
6. RMS thermal vibration

Occupancy currently is assumed to be 1 for all atomic sites. The x, y, z, and RMS thermal vibration magnitude (root-mean-square atomic displacement) are all in units of Angstroms. Typical values for the RMS thermal displacements are 0.05 to 0.1 Angstroms.

For this tutorial, please download these five .xyz files:

[Barium Neodymium Titanate - [0 1 0] zone axis](data/barium_neodymium_titanate_0_1_0.xyz)

[Barium Neodymium Titanate - [0 2 1] zone axis](data/barium_neodymium_titanate_0_2_1.xyz)

[Barium Neodymium Titanate - [0 1 1] zone axis](data/barium_neodymium_titanate_0_1_1.xyz)

[Barium Neodymium Titanate - [0 1 2] zone axis](data/barium_neodymium_titanate_0_1_2.xyz)

[Barium Neodymium Titanate - [0 0 1] zone axis](data/barium_neodymium_titanate_0_0_1.xyz)

After downloading these files, load the first file ([0 1 0] zone axis) into `Prismatic` using the **Load Coords** button. The below screenshot shows what you should see after loading this file, specifically the unit cell dimensions along x, y and z:

![SuperSTEM screenshot 01](img/SuperSTEM/screenshot01.png){:width="960"}



&nbsp;
<a name="step3"></a>
## 3 - Set microscope parameters and simulation settings.
text

&nbsp;
<a name="step4"></a>
## 4 - Calculate and view projected atomic potentials.
text

&nbsp;
<a name="step5"></a>
## 5 - Determine required unit cell tiling, examine PRISM accuracy.
text

&nbsp;
<a name="step6"></a>
## 6 - Run simulations using the PRISM algorithm.
text

&nbsp;
<a name="step7"></a>
## 7 - Save simulation results, save output images.
text

&nbsp;
<a name="step8"></a>
## 8 - Further simulations.
text








