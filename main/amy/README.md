# libAMY - additive music synthesizer library

`AMY` is a C library for additive sound synthesis. It is the synthesis engine behind [`alles`](https://github.com/bwhitman/alles). It was designed work on small, memory constrained MCUs like the ESP series, but can be ported to most any system with a FPU. We ship it with a Python wrapper so you can make nice sounds in Python, too.

`AMY`'s features include: 
 * Support for an arbitrary number of oscillators, each with:
   * Band-limited sine, saw, triangle, pulse waves powered by Dan Ellis' `libblosca`
   * Karplus-strong and noise synthesis 
   * PCM synthesis from a buffer
   * Bandpass, lowpass and hi-pass filters, with adjustable center frequency and resonance
   * 2 breakpoint generators that can control amplitude, frequency, pulse width or filter parameters
 * Any oscillator can modulate another using LFO modulation of amplitude, frequency, pulse width or filter parameters
 * Selectable FM-style algorithms for modulating frequency and mixing oscillators, DX7-like and somewhat compatible
 * Partial synthesis in the style of Alles or Atari's AMY

## Controlling AMY

AMY's wire protocol is a series of numbers delimited by ascii characters that define all possible parameters of an oscillator. This was a design decision to make using AMY from any sort of environment as easy as possible, with no data structure or parsing overhead on the client. It's also readable and compact, far more expressive than MIDI and can be sent over network links, UARTs, or as arguments to functions or commands. We send AMY messages over multicast UDP to power [`alles`](https://github.com/bwhitman/alles) and have generated AMY messages in C, C++, Python, Max/MSP, shell scripts, JavaScript and more. 

AMY accepts commands in ASCII, each command separated with a newline (you can group multiple messages in one, to avoid network overhead if that's your transport). Like so:

```
v0w4f440.0l0.9\n
```

AMY's full commandset:

```
a = amplitude, float 0-1+. use after a note on is triggered with velocity to adjust amplitude without re-triggering the note
A = breakpoint0, string, in commas, like 100,0.5,150,0.25,200,0 -- envelope generator with alternating time(ms) and ratio. last pair triggers on note off
B = breakpoint1, set the second breakpoint generator. see breakpoint0
b = feedback, float 0-1. use for the ALGO synthesis type or for karplus-strong 
d = duty cycle, float 0.001-0.999. duty cycle for pulse wave, default 0.5
D = debug, uint, 2-4. 2 shows queue sample, 3 shows oscillator data, 4 shows modified oscillator. will interrupt audio!
f = frequency, float 0-44100 (and above). default 0. Sampling rate of synth is 44,100Hz but higher numbers can be used for PCM waveforms
F = center frequency of biquad filter. default 0. 
g = modulation target mask. Which parameter modulation/LFO controls. 1=amp, 2=duty, 4=freq, 8=filter freq, 16=resonance. Can handle any combo, add them together
G = filter type. 0 = none (default.) 1 = low pass, 2 = band pass, 3 = hi pass. 
I = frequency ratio. used for ALGO types, where the base note frequency controls the modulators
L = modulation source oscillator. 0-63. Which oscillator is used as an modulation/LFO source for this oscillator. Source oscillator will be silent. 
l = velocity (amplitude), float 0-1+, >0 to trigger note on, 0 to trigger note off.  
n = midinote, uint, 0-127 (this will also set f). default 0
o = algorithm, choose which algorithm for the algorithm oscillator, uint, 0-31. mirrors DX7 algorithms
O = algorithn source oscillators, choose which oscillators make up the algorithm oscillator, like "0,1,2,3,4,5" for algorithm 0
p = patch, uint, 0-999, choose a preloaded PCM sample or DX7 patch number for FM waveforms. See patches.h, pcm.h. default 0
P = phase, float 0-1. where in the oscillator's cycle to start sampling from (also works on the PCM buffer). default 0
R = q factor / "resonance" of biquad filter. float. in practice, 0 to 10.0. default 0.7.
S = reset oscillator, uint 0-63 or for all oscillators, anything >63, which also resets speaker gain and EQ.
T = breakpoint0 target mask. Which parameter the breakpoints controls. 1=amp, 2=duty, 4=freq, 8=filter freq, 16=resonance. Can handle any combo, add them together. Add 32 to indicate linear ramp, otherwise exponential
t = time, int64: ms since some fixed start point on your host. you should always give this if you can.
v = oscillator, uint, 0 to 63. default: 0
V = volume, float 0 to about 10 in practice. volume knob for the entire synth / speaker. default 1.0
w = waveform, uint, 0 to 9 [SINE, SQUARE, SAW, TRIANGLE, NOISE, FM, KS, PCM, ALGO, OFF]. default: 0/SINE
W = breakpoint1 target mask. 
x = "low" EQ amount for the entire synth (Fc=800Hz). float, in dB, -15 to 15. 0 is off. default: 0
y = "mid" EQ amount for the entire synth (Fc=2500Hz). float, in dB, -15 to 15. 0 is off. default: 0
z = "high" EQ amount for the entire synth (Fc=7500Hz). float, in dB, -15 to 15. 0 is off. default: 0
```

Synthesizer state is held per oscillator, so you can optionally send only changes in parameters each message per oscillator. Oscillators don't make noise until velocity (`l`) is over 0, you can consider this a "note on" and will trigger envelopes / modulation if set.

You can use any environment to pass AMY commands to the synthesizer and retrieve rendered audio samples back using a very simple API:

```c
#include "amy.h"

int16_t * hello_AMY() {
	start_amy(); // initialize the oscillators and sequencer
	parse_message("v0f440.0w0l0.5t100\n"); // start rendering a 440Hz sine wave on oscillator 0 at 100ms
	return fill_audio_buffer_task(); // render BLOCK_SIZE (128) samples of S16LE ints
}
```

libAMY ships with a Python module and the [`alles`](https://github.com/bwhitman/alles) project ships with demos and patches, alongside network features for that synth.

```python
import amy
amy.start()
amy.send("v0f440.0w0l0.5t100\n")
samples = amy.render(1.0) # seconds
amy.stop()
```

Using `amy.py`'s local mode:

```python
import amy, alles
amy.live() # starts an audio callback thread to play audio in real time
amy.send(osc=0,freq=3000,amp=1,wave=amy.SINE)
amy.send(osc=1,freq=500,amp=1,wave=amy.SINE,bp0_target=amy.TARGET_AMP,bp0="0,0,10,1,5000,0")
amy.send(osc=2,wave=amy.ALGO,algorithm=4,algo_source="0,1")
amy.note_on(osc=2,vel=1,freq=400) # play an FM bell tone
alles.drums() # play a drum pattern
amy.pause()
```


## Compile AMY and install the Python library

```
$ cd alles/main/amy
$ python3 setup.py install
$ python3 -m pip install sounddevice numpy # needed to render live audio in the shell
```


