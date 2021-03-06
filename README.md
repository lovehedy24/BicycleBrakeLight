
The goal of this project is to explore real world applications of accelerometers by designing a bicycle tail light that can light up when the bicycle is slowing down. There are commercial products that do this and at least according to their demo videos seem to work properly, so I have set on a quest to reproduce the device and get it to work in a satisfactory way.

The picture below shows the prototype used. The larger board is an Arduino Mini and the smaller one is an accelerometer breakout board. The brake light LED is the clear one visible on the bottom.

![Proto](Documentation/proto.jpg)

The same protobaord slightly modified and fit inside a bicycle tail light from which I removed the existing LEDs:

![Proto](Documentation/assembled_open.jpg)

And the final prototype assembled and ready to go. The power switch is on the back side, not visible here.

![Proto](Documentation/assembled.jpg)

There is a video available showing the device in action, it's unfortunately rather dark, I might shoot another one if I have time: http://youtu.be/WYHeaFL7syk

I have later replaced the two AAA batteries with a small LiPo and USB charging circuit and been using it now on my daily commute trips, which attracted the random comments of fellow cyclists expecially when stopping at road crossings.

Challenges
==========

The output of an accelerometer is far from clean and, even more so, when riding on rough road or off-road. A first naive attempt to get an idea was to just read acceleration in the direction of travel and light an LED when this was below zero or a certain threshold. Needless to say results were awful with the LED randomly blinking at every bump on the road.

Running Average Filter
==========

A running average is a smoothing process that is achieved by averaging the last n samples of a signal. This also has a low pass frequency response when seen in frequency domain. Unfortunately this type of filter doesn't have a very good frequency response, a rather slow roll off and a poor out of band rejection. I wanted to give it a try anyway as it's extremely simple to implement and it might be sufficient for this type of application. I tried to shoot first for a 1Hz 3dB cut-off frequency. The normalized frequency response for a running average filter of length N can be calculated as:

![Frequency Response](Documentation/RunningAverageFreqResp.gif)

For a filter length N=20 the normalized frequency response looks like this:

![Frequency Response](Documentation/RunningAverageFreqPlot.png)

So we can expect an out of band rejection of roughly 10dB. By calculating values for the above formula we also see that the -3dB cut-off happens at a normalized frequency of 0.03. So I choose a sampling frequency of 40Hz that places the cut-off at 1.2Hz with filter length N=20. The insertion delay in this case is around 0.5s. This is important to our application as it will affect the lag between the moment in which the bike starts to slow down and when the LED will light up.

Results were not bad in road tests with the LED lighting up at every brake except the weakest ones. Results were not excellent though and often potholes and rough roads caused the light to intermittently come up.

FIR Filter
============

A filter with a better frequency response is a FIR (Finite Impulse Response) filter. This is basically a convolution of signal samples with a set of pre-calculated values known as taps. As is often the case there is a trade-off between the filter length (amount of taps) and the frequency response. 

I kept the original target to shoot for a 1Hz low pass filter and used an online tool to calculate the filter taps (http://t-filter.appspot.com/fir/index.html). In an attempt to reduce the amount of taps I kept the sampling frequency low at 10Hz so that I could get the 1Hz low pass with just 15 taps. The frequency response as plotted on the mentioned site can be seen below:

![Frequency Response](Documentation/FIRFilterResponse.png)

There are two issues with this filter. The first is the quite high ripple in-band, that is more than 2dB. This means that calibration will be rather off depending on the frequency of the acceleration components. The second problem is the rather low sampling frequency which will mean that any vibration from the road or bike parts above 5Hz will come back aliased in-band, and I can see lot of stuff moving above 5Hz on a bicycle. I decided to take it for a spin anyhow to see how it performed. It was much better than the running average one, as could be expected. At lower speeds there were no spurious LED blinks and modest brake efforts were detected. At higher speed though the LED started to act quite erratically, most likely because of higher frequency components aliasing.

Back to the drawing board I got another filter designed with 50Hz sampling frequency and just 0.5dB ripple with 45 taps. The amount of taps is, by the way, relevant not only because the micro-controller has limited RAM but also because the insertion delay of the filter is proportional to the filter length. To be more accurate is is equal to half of the taps count multiplied by the sampling interval. So for this filter the insertion delay is roughly 450mS. During road tests this proved to be much more reliable, but not perfect, the insertion delay also was noticeable. 

IIR filter the final design
============

So far I wanted mainly to get a feeling of how various filtering and smoothing options worked out, to work more towards the actual goal though a bit of research is needed. First of all from the datasheet of the accelerometer (MMA7361) we see it has a low pass filter on the outputs at around 400 Hz. So to avoid any aliasing we should choose a sampling frequency of at least 800 Hz, which would be possible but would make a filter for 1 Hz rather difficult. Fortunately on a moving bicycle there are not many strong high frequency vibrations as I could learn from this MIT study (http://web.mit.edu/2.tha/www/ppt/Bike-ISEA.pdf) where we see that frame vibrations in response to bumps peak at around 50Hz and are considerably weaker above 75 Hz. This still left with a needed sampling frequency of at least 150Hz. On the other hand considering only the lower 1Hz frequency portion will be of interest in this project the sample frequency can be lowered to 100 Hz as the, anyway weak, higher frequency components will alias back on the higher part of the spectrum and will be suppressed by the low pass filter. 

Now that the sample frequency was set it was clear that a FIR filter wouldn't have done as, to achieve the 1 Hz cut-off frequency it had to be rather long at 0.01 normalized frequency. So I went exploring IIR (Infinite Impulse Response) filters as I knew they can achieve the same performance of FIRs more efficiently. After playing with Octave a bit I was convinced and set for what became the final filter for this application a 1Hz 3rd order butterworth low pass. 

Road tests gave amazing results with very short delay in detecting the braking. Also potholes and rough road didn't give false readings at all.

Slope Compensation
============

So far tests were all carried out on level roads. Going down or uphill poses a problem though as part of the gravity goes to the forward motion axis adding or subtracting from the braking acceleration. For instance a 10% down slope would put 0.1g on the forward facing axis, which would be already detected as braking.

Research showed that this issue is usually solved with so called 6 axes accelerometers in which the usual 3 axes accelerometer is supplemented by a 3 axes gyroscope. Cheaper systems that make use of 3 axes accelerometers though estimate the gravity component by having two low pass filters one of which is set to a lower frequency to separate gravity. This is based on the assumption that motion acceleration changes faster than tilt. This is not true in all applications but in this case it seems reasonable as roads will change slope more slowly than the braking takes place.

After some tests a 0.1 Hz low pass filter seemed to give the best results so I went for that in final design. I had to make this a 2nd order one though as a 3rd order gave too small coefficients that were an issue due to the limited precision of floats on an Arduino. This was one more reason to move to IIR filters as this is a normalized cut-off frequency of 0.001!

Brake Detection
============

After testing a simple threshold detector I decided to have some hysteresis in the detector to avoid the light to shortly blink intermittently when the braking effort was near the threshold. At first I had roughly estimated a threshold of 0.1g. This in practice turned out to be a relatively vigorous brake effort so, experimentally, I dropped the threshold to 0.05g and the release threshold to 0.03g.  

Calibration
============

The chosen accelerometer type outputs the read values as an analog value, for this reason calibration is needed to convert such values, once sampled, to actual acceleration. For this system a simple calibration procedure has been chosen that uses earth gravity as reference. When the device is placed in calibration mode it will simply read all axes acceleration for 10 seconds and store the maximum and minimum values read for each axis. The operator is supposed to slowly rotate the device on all 6 sides so that each axis is exposed to accelerations of +1g to -1g. Of course this system is not optimal and requires to move the device slowly during calibration. Results can be verified after calibration ensuring +1g and -1g values are reported for all axes.

