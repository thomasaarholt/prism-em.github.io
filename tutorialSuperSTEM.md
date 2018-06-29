

![Prismatic screenshot 01](img/SuperSTEMtopbar.png){:width="1184px"}


## Table of Contents  
- [1 - Install `Prismatic`](#step1)
- [2 - Load atomic coordinates](#step2)
- [3 - Set microscope parameters](#step3)
- [4 - Projected atomic potentials](#step4)
- [5 - Unit cell tiling and `PRISM` accuracy](#step5)
- [6 - Run simulations](#step6)
- [7 - Save results](#step7)
- [8 - Further simulations](#step8)




&nbsp;
<a name="step1"></a>
## 1 - Download and Install the `Prismatic` GUI

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

The GPU portions of `Prismatic` were developed using CUDA. This means that Apple laptops and Mac desktops running AMD GPUs cannot make use of CUDA GPU code. However, the PRISM algorithm is quite fast. Therefore you may be able to run STEM simulations on Apple laptops in reasonable times. After downloading and installing `Prismatic`, run it to verify the installation has succeeded.



&nbsp;
<a name="step2s"></a>
## 2 - Download atomic coordinates, load into `Prismatic`


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

You should see x, y and z cell dimensions of 22.3, 12.2, and 7.7 Angstroms respectively.









&nbsp;
<a name="step3"></a>
## 3 - Set microscope parameters and simulation settings.

In the following sections, I have highlighted each `Prismatic` setting using <span style="color:red">**bold red text**</span>. This is to ensure that you can easily find the parameters used in this tutorial.

### Sample Settings:

Besides loading and saving, this settings box shows the cell dimensions and allows us to tile the unit cell along the three primary dimensions. Initially, we are going to skip over the **Tile Cells** settings, because they require very careful analysis to set correctly.

### Simulation Settings Block 1:

The first two buttons, **Load Parameters** and **Save Parameters** can save and load text files containing `Prismatic` settings. These files can be used in the GUI or command line versions of `Prismatic` and thus are very useful when submitting batch simulations.

The first setting, **Pixel Size**, is extremely important because it controls the high angle scattering accuracy of the simulation. This is because the pixel size <img src="https://latex.codecogs.com/svg.latex?\Large&space;p"/>
 in real space defines the maximum inverse spatial coordinate value in Fourier space <img src="https://latex.codecogs.com/svg.latex?\Large&space;q_{max}"/>, using the formula:

<img src="https://latex.codecogs.com/svg.latex?\Large&space;q_{max}=\frac{1}{2 p}"/>

`Prismatic` also includes a mathematical function to prevent an artifact called "aliasing" from occurring, which happens because of the periodicity assumption of the discrete Fourier transform. `Prismatic` uses an "anti-aliasing" aperture at 0.5 times the maximum scattering angle, which further reduces the maximum effective scattering angle by a factor of 0.5. This fact combined with the above equation and the formula 
<img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha = \lambda q "/> gives the maximum scattering angle <img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha_{max}"/> equation


<img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha_{max} = \frac{\lambda}{4 p}"/>

where <img src="https://latex.codecogs.com/svg.latex?\Large&space;\lambda "/>  is the relativistic electron wavelength. As an example, consider a case where we want to simulate 300 kV electron scattering angles up to 100 mrads. At 300 kV, <img src="https://latex.codecogs.com/svg.latex?\Large&space;\lambda \approx "/> 0.02 Angstroms. Plugging these values into the above equation gives

<img src="https://latex.codecogs.com/svg.latex?\Large&space;p = \frac{\lambda}{4 \alpha_{max}} = \frac{0.02}{(4)(0.1)} = 0.05"/>

i.e. we require a 0.05 Angstrom pixel size <img src="https://latex.codecogs.com/svg.latex?\Large&space;p"/> in order to reach a maximum scattering angle <img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha_{max}"/> of 100 mrads.



For this tutorial, we will use a **Pixel Size** value of <span style="color:red">**0.1 Angstroms**</span> in order to save time, and we will set the accelerating voltage (in kV) using the **Energy** box to <span style="color:red">**100 kV**</span>. Note that including these settings will immediately update the wavelength <img src="https://latex.codecogs.com/svg.latex?\Large&space;\lambda"/> and maximum scattering angle <img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha_{max}"/> displayed values to

<img src="https://latex.codecogs.com/svg.latex?\Large&space;\lambda = "/> 0.037 Angstroms

<img src="https://latex.codecogs.com/svg.latex?\Large&space;q_{max} = "/> 92.5 mrads

The next value  **Potential Bound** specifies how far from the atomic core we will integrate the projected potentials. Set this value to <span style="color:red">**2 Angstroms**</span> for reasonable accuracy.




### Simulation Settings Block 2:

The next set of parameters describe the values required to describe the incident converged electron probe. The semiangle of the probe specified by a condenser aperture is given by the **Probe Semiangle** box, which you should set to <span style="color:red">**30 mrads**</span>. For the `PRISM` algorithm we also need to specify the **Probe <img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha"/> limit**, which corresponds to the maximum scattering angle computed for the Compact S-Matrix. This value should be set slightly larger than the **Probe Semiangle**, so we will use a value of <span style="color:red">**32 mrads**</span> for the **Probe <img src="https://latex.codecogs.com/svg.latex?\Large&space;\alpha"/> limit**. 

The next three values are **C1**, **C3**, and **C5**, corresponding to defocus, and the first two orders of spherical aberration respectively. The convention used in `Prismatic` is a probe defined by

<img src="https://latex.codecogs.com/svg.latex?\Large&space;\Psi(\vec{q}\,) = \exp[ -i \chi (\vec{q}\,)]"/> 

where

<img src="https://latex.codecogs.com/svg.latex?\Large&space;\chi(\vec{q}\,) = \pi \lambda |\vec{q}\,|^2 C_1 + \frac{\pi}{2} \lambda^3 |\vec{q}\,|^4 C_3 + \frac{\pi}{3} \lambda^5 |\vec{q}\,|^6 C_5"/> 

To keep this tutorial simple, we will assume an ideal microscope with no aberrations, where <span style="color:red">**C_1 = C_3 = C_5 = 0**</span>.

The next parameter value on the right side is the **Random Seed**, which is only used to make the simulation more repeatable for testing - thus we do not care what this value is set to for the purposes of this tutorial. The next value is the **# of FP**, which means "number of frozen phonon (FP) configurations." These multiple FP configurations are used to average the scattering pathways due to thermal motion of the atoms over multiple simulations. All high accuracy simulations should use multiple FPs, and typically the more the better. I recommend using (at a minimum) 8 FPs for thick samples, and 32 FPs for thin samples. For this tutorial, we will use <span style="color:red">**1 FP**</span> configuration.

The next value is **Slice Thickness** which defines the spacing of potential slices used in the atomic potentials. A smaller value here will make the simulation more accurate, while a larger value will speed up the simulation. For this simulation we will use a typical value of <span style="color:red">**2 Angstroms**</span>. The last value in this column is **Detector Angle Step**, which is used for the 3D simulation output to specify the width of each annular bin used in the output integration. The 3D output is a series of concentric rings for different scattering angles, recorded for each probe position. This value should be small enough to allow any necessary virtual detector to be constructed from the simulation. We will use the default value of <span style="color:red">**1 mrad**</span>.



### Simulation Settings Block 3:

The final group of simulation settings deals with the `PRISM` interpolation factors, the STEM probe positions and tilts. We will be changing these settings later, but for now set **PRISM Interpolation Factors** to <span style="color:red">**4**</span> for both x and y.  The spacing of STEM probes is set by the **Probe Step** box, which we will set to <span style="color:red">**0.2 Angstroms**</span>. **Probe Tilt** should be set to <span style="color:red">**0 mrads**</span> for both x and y, and the **Scan Window** should run from <span style="color:red">**0 to 0.9999**</span> for both x and y. This will create a square grid of STEM probes that will cover the entire field of view. The reason to not set the maximum range to 1.0 is that if the probe step can be evenly divided into the simulation cell size, we would duplicate the probe located at 0.0 with the probe located at 1.0, due to the periodicity of the simulation.

Note that it is very important to remember the difference between **Pixel Size** and **Probe Step** settings, both in units of Angstroms. The **Pixel Size** setting refers to the sampling of pixels for the atomic potentials, the STEM probes, and the various numerical operator functions used during the simulation. By contrast, **Probe Step** refers only to the spacing of the STEM probes, and no other parts of the simulation. Later in the tutorial, we will for example run simulations consisting of only single STEM probe positions.

Now that you have entered the basic settings, your window should look like so:

![SuperSTEM screenshot 02](img/SuperSTEM/screenshot02.png){:width="898"}




&nbsp;
<a name="step4"></a>
## 4 - Calculate and view projected atomic potentials.
text

&nbsp;
<a name="step5"></a>
## 5 - Determine required unit cell tiling, examine `PRISM` accuracy.
text

&nbsp;
<a name="step6"></a>
## 6 - Run simulations using the `PRISM` algorithm.
text

&nbsp;
<a name="step7"></a>
## 7 - Save simulation results, save output images.
text

&nbsp;
<a name="step8"></a>
## 8 - Further simulations.
text








