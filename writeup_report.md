#  Radar Target Generation and Detection

---


## [Rubric](https://review.udacity.com/#!/rubrics/2548/view) Points


[//]: # (Image References)

[image1]: ./writeup_images/FFT1.png "FFT1"
[image2]: ./writeup_images/FFT2.png "FFT2"
[image3]: ./writeup_images/2D_CFAR.png "2D_CFAR"



---

### FMCW Waveform Design

#### Criteria

Using the given system requirements, design a FMCW waveform. Find its Bandwidth (B), chirp time (Tchirp) and slope of the chirp.

#### Meets Specifications

For given system requirements the calculated slope should be around 2e13

#### Solution

The solution uses 

```octave

c = 3e8;
range_res = 1;
range_max = 200;

B = 3e8 / (2 * range_res)
Tchirp = 5.5 * 2* range_max / c
Slope = B/Tchirp

```

### Simulation Loop

#### Criteria

Simulate Target movement and calculate the beat or mixed signal for every timestamp.

#### Meets Specifications

A beat signal should be generated such that once range FFT implemented, it gives the correct range i.e the initial position of target assigned with an error margin of +/- 10 meters.

#### Solution


```octave

for i=1:length(t)         
    
    %For each time stamp update the Range of the Target for constant velocity. 
    
    r_t(i) = R + V * t(i);
    
    %For each time sample we need update the transmitted and received signal. 
    
    % The time delay is the time needed to reach the current position of the target and come back to the receiver.
    td(i) = 2 * r_t(i) / c; 
    
    Tx(i) = cos(2 * pi * (fc * t(i) + Slope * t(i)^2 / 2));
    Rx(i) = cos(2 * pi * (fc * (t(i) - td(i)) + Slope * (t(i) - td(i))^2 / 2));    
    
    %Mxing the Transmit and Receive to generate the beat signal
    Mix(i) = Tx(i).*Rx(i);
¡  
end

```

### Range FFT (1st FFT)

#### Criteria

Implement the Range FFT on the Beat or Mixed Signal and plot the result.

#### Meets Specifications

A correct implementation should generate a peak at the correct range, i.e the initial position of target assigned with an error margin of +/- 10 meters.

#### Solution

```octave

%reshape the vector into Nr*Nd array. Nr and Nd here would also define the size of Range and Doppler FFT respectively.

signal =reshape(Mix, [Nr, Nd]);

%run the FFT on the beat signal along the range bins dimension (Nr) and normalize.

signal_fft = fft(signal,Nr)./Nr;

% Take the absolute value of FFT output

signal_fft = abs(signal_fft);

% The utput of the FFT is a double sided signal
% We discard half of the samples.

signal_fft  = signal_fft(1:Nr/2+1);

```
Result of the 1D FFT

![alt text][image1]

Result of the 2D (Range and Doppler) FFT

![alt text][image2]

### 2D CFAR

#### Criteria

Implement the 2D CFAR process on the output of 2D FFT operation, i.e the Range Doppler Map.

#### Meets Specifications

The 2D CFAR processing should be able to suppress the noise and separate the target signal. The output should match the image shared in walkthrough.

#### Solution

The solution is based on two loops to iterate over the range and the Doppler dimensions of the RDM array. The noise level sum was implemented using the built-in "sum" function and array slice indexing to specify the values to be added. 

The number of cells added, needed to calculate the average for the threshold, is based directly on the concept presented in the lesson. The values for the training and guarding cell numbers, as weel ass the SNR offset were selected based on initial values presented in the lessons and fine tuned after a few iterations to achieve robust target detection as shown in the resulting plot.

```octave


Tr = 12;
Td = 6;

Gr = 6;
Gd = 2;

offset = 24;

for i = Gr + Tr+ 1: (Nr/2) -(Gr + Tr+ 1)
    noise_level = 0;
    for j= Gd + Td +1:Nd -(Gd + Td +1)  
        noise_level = sum(db2pow(RDM(i-Gr-Tr:i-Gr,j-Gd-Td:j-Gd)))+ sum(db2pow(RDM(i+Gr:i+Gr+Tr,j+Gd:j+Gd+Td)));
        threshold = pow2db(noise_level/((2*Tr+2*Gr+1)*(2*Td+2*Gd+1)-(2*Gr+1)*(2*Gd+1))) + offset;
        
        if RDM(i,j) > threshold
            RDM(i,j) = 1;
        else
            RDM(i,j) = 0;
        end
    end
end

```

An alternative approach to sum the training cells noise level without using the "sum"function was also explored.

```octave
for i = Gr + Tr+ 1: (Nr/2) -(Gr + Tr+ 1)
    for j= Gd + Td +1:Nd -(Gd + Td +1)         
         noise_level = 0;
         for r=1:Tr
             for d = 1:Td
                 noise_level = noise_level + db2pow(RDM(i-Gr-r,j-Gd-d))+ db2pow(RDM(i+Gr+r,j+Gd+d));
             end
         end
        threshold = pow2db(noise_level/((2*Tr+2*Gr+1)*(2*Td+2*Gd+1)-(2*Gr+1)*(2*Gd+1)))+offset;
        
        if RDM(i,j) > threshold
            RDM(i,j) = 1;
        else
            RDM(i,j) = 0;
        end
    end
end

```

Supression of the non-thresholded cells at the edges was achieved using array slice indexing. The slicing indexes of each edge were determined based on the training and guarding cells for each dimension (range and Doppler).

```octave

RDM(1:Nr/2,1:Gd+Td) = 0;
RDM(1:Nr/2,Nd-(Gd+Td):Nd) = 0;
RDM(1:Gr+Tr,Td+Gd:Nd-(Gd+Td)) = 0;
RDM(Nr/2-(Gr+Tr):Nr/2,Td+Gd:Nd-(Gd+Td)) = 0;

```
Result of 2D CFAR
![alt text][image3]

