# Linux-Audio-Loopback-Device

This is a tutorial on creating audio loopback devices on linux. For this we use the following technologies which are I suppose readily available on Ubuntu 18 which is the platform on which this was tried:
1) ALSA Loopback module 'snd-aloop'
2) Gstreamer 'gst-launch-1.0'


## Initialization

To make a loopback device we will first need to load the ALSA loopback module. We can do that by:
```
sudo modprobe snd-aloop
```

The following command will give us the list of available playback device - output devices like the speaker that are writable.
```
$ aplay -l
**** List of PLAYBACK Hardware Devices ****
card 0: HDMI [HDA Intel HDMI], device 3: HDMI 0 [HDMI 0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 7: HDMI 1 [HDMI 1]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 8: HDMI 2 [HDMI 2]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 9: HDMI 3 [HDMI 3]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 0: HDMI [HDA Intel HDMI], device 10: HDMI 4 [HDMI 4]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: PCH [HDA Intel PCH], device 0: ALC3232 Analog [ALC3232 Analog]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 2: Loopback [Loopback], device 0: Loopback PCM [Loopback PCM]
  Subdevices: 7/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 2: Loopback [Loopback], device 1: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
```

We can see from the above output that card0 and card1 are hardware devices whereas card2 is Loopback. Similarly for listing input devices - devices like the microphone - we use the following command:
```
$ arecord -l
**** List of CAPTURE Hardware Devices ****
card 1: PCH [HDA Intel PCH], device 0: ALC3232 Analog [ALC3232 Analog]
  Subdevices: 0/1
  Subdevice #0: subdevice #0
card 2: Loopback [Loopback], device 0: Loopback PCM [Loopback PCM]
  Subdevices: 7/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 2: Loopback [Loopback], device 1: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
```

We can see that card1 has the hardware microhone whereas card2 is our loopback.



## How does snd-aloop or ALSA Loopback module work?

Putting it very simply, ALSA provides us with two virtual devices, one of them is a speaker - a device you can write to - and the other is a microphone - a device you can read from - and both of them are connected so that whatever you write on the speaker device can be read from the microhphone device. We can call this a connected pair of mic and speaker. ALSA provides us with 8 such pairs.

In ALSA land a single audio device is identified by a device name that has the pattern 'hw:x,y,z' where 'x' is the card number, 'y' is the device number, and 'z' is the subdevice number. So in our example above the loopback module gives us a card 'card2'
, it has two devices 'device 0' and 'device 1', and each of the devices have 8 subdevices 0-7. So coming back to how the loopback module works, all the connected speaker-mic pairs would be: 
```
'Speaker'   ====> 'Mic'
'hw:2,1,0'  ====> 'hw:2,0,0'
'hw:2,1,1'  ====> 'hw:2,0,1'
'hw:2,1,2'  ====> 'hw:2,0,2'
'hw:2,1,3'  ====> 'hw:2,0,3'
'hw:2,1,4'  ====> 'hw:2,0,4'
'hw:2,1,5'  ====> 'hw:2,0,5'
'hw:2,1,6'  ====> 'hw:2,0,6'
'hw:2,1,7'  ====> 'hw:2,0,7'
```
NOTE: All the subdevices in 'device 0' are Mics/Srcs - readable - and all those in 'device 1' are Speakers/Sinks - writable. Which means that anything written on 'hw:2,1,n' would be read on 'hw:2,0,n' where n is between 0-7 inclusive.




## Experiment:

For a little experiment we will be writing a test audio stream to one of the loopback sinks and then reading the associated loopback source and writing to our hardware speakers. For this we will use gstreamer.

1) first we load the loopback module
```
$ sudo modprobe snd-aloop
```

2) Write test stream to the loopback speaker
```
$ gst-launch-1.0 audiotestsrc ! alsasink device="hw:2,1,3"
```

2) Read test stream from loopback mic and play on speakers
```
$ gst-launch-1.0 alsasrc device="hw:2,0,3" ! alsasink
```

What this experiment demonstrates is that what is written to one end of the loopback module can be read from the other end. This could be used for softwares like voice changers, a pipeline for that would be that you read from the hardware mic, apply some affects and write the resulting stuff to loopback sink, then set the loopback mic as the default microphone, this way any applications that read from the mic will be reading the distorted audio from the loopback mic.
