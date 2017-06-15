# About *Prismatic*
*Prismatic* is a CUDA/C++ software package for fast image simulation in scanning transmission electron microscopy (STEM). It includes parallel, data-streaming implementations of both the plane-wave reciprocal-space interpolated scattering matrix (PRISM) and multislice algorithms using multicore CPUs and CUDA-enabled GPU(s), in some cases achieving accelerations as high as 1000x or more relative to traditional methods. *Prismatic* is fast, free, open-sourced, and contains a graphical user interface.

## File Formats

### Input

Input files containing the atomic coordinates should be in the following XYZ format.

* 1 comment line
* 1 line with the 3 unit cell dimensions a,b,c (in Angstroms)
* *N* lines of the form Z, x, y, z, occ, thermal_sigma where Z is the atomic number, x, y, and z are the coordinates of the atom within the unit cell (in Angstroms), occ is the occupancy number (almost always 1, and currently not used in PRISM), and thermal_sigma is the standard deviation of random thermal motion (Debye-Waller effect).
* -1 to indicate end of file

Here is a sample file, "SI100.XYZ"

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

Outputs are written to binary [.mrc](http://bio3d.colorado.edu/imod/doc/mrc_format.txt) files with float-precision. There are potentially 2D, 3D, and 4D output. For all outputs, there are two dimensions corresponding to X and Y probe positions. At each position, *Prismatic* can output the full probe (4D), a radially integrated ouput into virtual detector bins (3D), or further integrated over a range of detector bins to produce a single value for each scan position (2D). The 3D output is considered to be the primary result and is the only output produced by default; however, any combination of 2D, 3D, and 4D outputs may be produced with a single simulation. The metadata parameter `filename_output`, set with "-o" at the command line, is used as a base filename and modified depending on the output type.

* 2D output: Produced by adding the command line option "-2D ang\_min ang\_max" where "ang\_min" and "ang\_max" are the inner and outer integration angles in mrad. The resulting image will be composed as either "prism\_2Doutput\_" + `filename_output` or "multislice\_2Doutput\_" + `filename_output`. By default this is off.
* 3D output: Controlled by command line option "-3D 0/1" where 0 or 1 is a boolean on/off. The 3D output is saved to `filename_output`. By default this is on.
* 4D output: Controlled by command line option "-4D 0/1" where 0 or 1 is a boolean on/off. The 4D output saves a separate 2D MRC image for each X/Y probe scan position. The output name for each image is tagged with the X and Y scan position index + `filename_output`. By default this is off.

For example, the `prism` command 
`prism -i atoms.XYZ -2D 0 10 -4D 1 -3D 1 -o example.mrc`
will produce "prism\_2Doutput\_example.mrc" with the 2D bright field image integrated from 0-10 mrad, the 3D output in "example.mrc", and many 2D images of the form "example\_X##\_Y##\_FP##.mrc". For the time being, it will likely require some scripting on the user's part to wrangle the 4D output. In the future, we intend to introduce an hdf5 format to contain each of these outputs in a unified way. 
