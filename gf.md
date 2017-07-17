# Tutorial

This tutorial will walk you through your first GENFIRE 3D reconstruction. You will 
simulate a reconstruction of a tomographically-acquired tilt series using a vesicle model 
with GENFIRE's GUI.

First thing's first -- open the GUI. It looks like this

![GENFIRE GUI](img/gui.png)

This is the main window for running GENFIRE reconstructions. Here you can select the filenames
containing the projection images, Euler angles, 3D support and set reconstruction parameters
like the number of iterations to run and the oversampling ratio (the amount of zero padding to add
to the projections prior to gridding). First we have to create a simulated dataset so that we have
something to work with. For that we can use the projection calculator, which can be accessed from a drop-down menu at the top of the screen:

	Projection Calculator -> Launch Projection Calculator

You should now have a blank instance of the projection calculator, like this:

![The GENFIRE Projection Calculator Module](img/ProjectionCalculator_blank.png)

Now we need to select a 3D model. Click Browse, find "vesicle.mrc" in the data
directory of the GENFIRE source code, then click open. You will be prompted to select
an oversampling ratio. The oversampling ratio controls the amount of zero padding applied 
to the model -- specifically the oversampling ratio is the total array size divided
by the size of the object. The purpose of this zero-padding is to increase the accuracy
of the projection calculation. The tradeoff is that larger oversampling ratios mean the
array size is bigger, and, therefore, slower. I find that an oversampling ratio of 3
is a good choice. Click OK, and GENFIRE will load the model, pad it,
compute the 3D FFT, and construct a linear interpolator. Once finished projections 
may be calculated relatively quickly.

Once loaded the zero-degree projection of the model will appear in the display.

![The GENFIRE Projection Calculator Module with Model](img/ProjectionCalculator_modelLoaded.png)

At this point you can adjust the Euler angles to explore what different views of the
model look like. Note that these are projection images, not surface renderings. If you 
are new to tomography, take a moment to explore how the projection images change as you
adjust the angles, in particular theta. This can give you some really nice intuition as 
to how 3D information is encoded in the 2D projection series.<br>
Once you are ready, calculate a projection image dataset from this model by clicking "Calculate Projection
Series from Model"

![Dialog to specify Euler angles -- empty](img/CalculateProjection_dialog_empty.png)

From this dialog you can specify the Euler angles for each of the calculated projections. 
To accomplish this you have two options.

1. Provide the Euler angles as a space-delimited .txt file where each 
row corresponds to one projection and provides the Euler angles as phi theta psi.
If you are confused about this format you can view the outputted file with option 2 to see an example.
Note there is no limitation on the angles for GENFINRE like there are in many single-axis tomography
reconstruction techniques, so you can use whatever you'd like.

2. Specify a single-axis tilt series. Specify the tilt angle, theta, 
as start = 0, step = 2, stop = 180 to calculate 91 equally spaced projections with no missing wedge.
Choose an output filename for the projection, make sure "Save Angles" is checked, 
then click "Calculate Projections" to perform the calculation. 


![Dialog to specify Euler angles -- ready](img/CalculateProjection_dialog_ready.png)

The calculation runs in the background on a separate thread. Once it is finished you will hopefully see
a success message like this

![GENFIRE GUI ready to reconstruct(img/gui_ready.png)

Note that the file created containing the Euler angles is the same name as the corresponding
projections with "_euler_angles" appended, in case you want an example of how to format
your own angle files.

For now, we will just use the default reconstruction parameters (more detail is given on them HERE).
Verify that the filenames of your data are correct, then start the reconstruction
by clicking the enormous green button.

![Reconstruction finished](img/gui_finished.png)

Congratulations, you have completed your first GENFIRE reconstruction! You can now view 
the error curves and a simple visualization of the results by clicking "Summarize Results"
and selecting the file with your results.

![Summary of results](img/summarize_results.png)

What's all this, you ask?


The left figure shows projection images of the reconstruction along the 3 principal 
axes and central slices. You'll be able to visualize the volume more closely in a moment.
The top error curve plots the total reciprocal error vs iteration number. This is the R-factor
between the FFT of the reconstruction and the entire constraint set. By default the reconstruction
is performed using resolution extension/suppression, so for the early iterations only the lowest
resolution constraints are enforced, but the error is still compared to all constraints so there 
are dips each time the constraint set is updated. This style of constraint enforcement is useful 
for noisy data -- here we have a noiseless simulation so you won't see much difference in the 
reconstruction if you turn it off.


The middle and bottom curves summarize the results for R-free. GENFIRE implements a modified version 
of the concept of R-free from X-ray crystallography. First, the constraint set is divided up into
bins (10 by default). In each spatial frequency bin, 5% of the values are withheld from the reconstruction.
At each iteration, the R-factor is calculated between the voxels in reciprocal space and these withheld values.
The purpose of this is a metric for prevention of overfitting to the data. Low values of R-free indicate
that recovered values for missing datapoints match the (withheld) input data, and by extension 
suggests confidence in reconstructed values where there is no measured datapoint to compare.<br><br>
The middle curve shows the mean value of R-free across all resolutions at each iteration. For clean
data it will generally mirror the reciprocal error curve. The bottom curve shows the value of R-free for
each spatial frequency bin at the final iteration. It generally increases with spatial frequency. For this
noiseless simulation the values are quite low, but for noisy data R-free will be higher. It is important
to remember that high values of R-free are not necessarily bad, they simply mean there is difference between
the recovered and measured reciprocal-space data. For noisy data this may be what you want, as resolution
extension/suppression can act as a denoising technique. However, R-free will also be high if your data
is not good. This illustrates the importance of considering multiple metrics when drawing conclusions about 
your results. Remember - "Garbage in, garbage out".

To explore your reconstruction, open the *Volume Slicer*

	Volume Slicer -> Launch Volume Slicer

and select your results.

![Volume Slicer](img/volume_slicer.png)

Here you can view individual layers of your reconstruction (or any volume) along the 3 principal directions.
You can also use this module to view your calculated projections.

Hopefully this tutorial has been helpful. Happy reconstructing!
