---
layout: post
title: Using aubio and alsaaudio with Python
date: 2013-02-09 15:58:04.000000000 +00:00
type: post
published: true
status: publish
categories:
- Python
tags: []
meta:
  _edit_last: '5947599'
  _wpas_skip_2804660: '1'
  _wpas_skip_2181874: '1'
  _oembed_d94cee0d33b8e6caebf412b841ae1af4: "{{unknown}}"
  _oembed_196ff3cb27f2c966ba6e8cf5aea5e730: "{{unknown}}"
  _oembed_a4911ad4bbed44d8cdcc6d30663d233d: "{{unknown}}"
  geo_public: '0'
  _publicize_job_id: '28979115902'
  _oembed_67c8576b927e02fbf2ad2c1a6080ccbc: "{{unknown}}"
  _oembed_e53a1e7f75645c98a657c2db4a093d33: "{{unknown}}"
  _oembed_c77e61c0ee60f75e49d97ae51a480796: "{{unknown}}"
  _oembed_af19aa321df642df6714f2f93bad1495: "{{unknown}}"
  _oembed_dc92171ae5e36c2ee6bf2c0aa4790b8d: "{{unknown}}"
  _oembed_5a406b421935d08c0d71c402d2afcdd4: "{{unknown}}"
author:
  login: chrisbaume
  email: chrisbaume@gmail.com
  display_name: chrisbaume
  first_name: ''
  last_name: ''
---
<span style="color:red">The code used in this post no longer works. For an up-to-date example, please see
[demo_alsa.py](https://github.com/aubio/aubio/blob/master/python/demos/demo_alsa.py).</span>

Aubio is an audio analysis library which contains implementations of some useful algorithms, including pitch detection.
It can be used with Python (through SWIG), but the [documentation](http://aubio.org/doc/files.html) is very light and
there doesn’t appear to be any
Python-specific instructions.

Below is a small program that listens to the default audio input using alsaaudio, finds the pitch and energy of the
signal using aubio, and prints the results to stdout. For the program to work, you will need the aubio and alsaaudio
Python libraries installed, which can be done in Ubuntu/Debian with the following command:

{% highlight shell %}
sudo apt-get install python python-alsaaudio python-aubio
{% endhighlight %}

The smpl_t data type referred to in the code can be replaced by Python’s float type, but the fvec_t type must be
populated one-by-one using the fvec_write_sample function.

{% highlight python %}
import alsaaudio, struct
from aubio.task import *
 
# constants
CHANNELS    = 1
INFORMAT    = alsaaudio.PCM_FORMAT_FLOAT_LE
RATE        = 44100
FRAMESIZE   = 1024
PITCHALG    = aubio_pitch_yin
PITCHOUT    = aubio_pitchm_freq
 
# set up audio input
recorder=alsaaudio.PCM(type=alsaaudio.PCM_CAPTURE)
recorder.setchannels(CHANNELS)
recorder.setrate(RATE)
recorder.setformat(INFORMAT)
recorder.setperiodsize(FRAMESIZE)
 
# set up pitch detect
detect = new_aubio_pitchdetection(FRAMESIZE,FRAMESIZE/2,CHANNELS,
                                  RATE,PITCHALG,PITCHOUT)
buf = new_fvec(FRAMESIZE,CHANNELS)
 
# main loop
runflag = 1
while runflag:
 
  # read data from audio input
  [length, data]=recorder.read()
 
  # convert to an array of floats
  floats = struct.unpack('f'*FRAMESIZE,data)
 
  # copy floats into structure
  for i in range(len(floats)):
    fvec_write_sample(buf, floats[i], 0, i)
 
  # find pitch of audio frame
  freq = aubio_pitchdetection(detect,buf)
 
  # find energy of audio frame
  energy = vec_local_energy(buf)
 
  print "{:10.4f} {:10.4f}".format(freq,energy)
{% endhighlight %}
