# alles

![picture](https://raw.githubusercontent.com/bwhitman/synthserver/master/pics/IMG_2872.jpeg)

Turns an ESP32 & an i2s chip & speaker into a WiFi controlled battery powered 10-voice synthesizer. Supports additive sine, saw, square, triangle oscillators as well as a full FM stage, modeled after the DX7 including support for DX7 patches. They're cheap to make ($7 for the ESP, $6 for the I2S amp, speakers from $0.50 up depending on quality). And only a few soldering points. 

The synthesizers listen to UDP multicast messages. The idea is you can install a bunch of them throughout a space and make a distributed / spatial version of the [Alles Machine](https://en.wikipedia.org/wiki/Bell_Labs_Digital_Synthesizer) / [AMY](https://www.atarimax.com/jindroush.atari.org/achamy.html) additive synthesizer where each speaker represents up to 10 partials, all controlled as a group or individually from a laptop or phone or etc. 

## Putting it together 

currently using

* [ESP32 dev board (any one will do, but you want pins broken out)](https://www.amazon.com/gp/product/B07Q576VWZ/) (pack of 2, $7.45 each)
* [The Adafruit I2S mono amplifier](https://www.adafruit.com/product/3006) ($5.95)
* [4 ohm speaker, this one is especially nice](https://www.parts-express.com/peerless-by-tymphany-tc6fd00-04-2-full-range-paper-cone-woofer-4-ohm--264-1126?gclid=EAIaIQobChMIwcX3-vXi5wIVgpOzCh0a7gjuEAYYASABEgLwf_D_BwE) ($9.77, but you can save a lot of money here going lower-end if you're ok with the sound quality)

### Power 

A 5V input (USB battery, USB input, rechargeable batteries direct to power input) powers both boards and speaker at pretty good volumes. A 3.7V LiPo battery will also work, but note the I2S amp will not get as loud (without distorting) if you give it 3.7V. I recommend using a USB battery pack that does not do [low current shutoff](https://www.element14.com/community/groups/test-and-measurement/blog/2018/10/15/on-using-a-usb-battery-for-a-portable-project-power-supply). I lucked on [this one at Amazon for $9.44](https://www.amazon.com/gp/product/B00MWU1GGI). The draw of the whole unit at loud volumes is around 150mA, so my battery should power a single synth making sound for 30 hours. 

### Wiring

Wire it up like this (I2S -> ESP)

```
LRC -> D25 / A1
BCLK -> D26 / A0
DIN -> D4 / A5
GAIN -> I2S Vin (i jumper this on the I2S board)
SD -> not connected
GND -> GND
Vin -> Vin / USB / 3.3 (or direct to your 5V power source)
Speaker connectors -> speaker
```

![closeup](https://raw.githubusercontent.com/bwhitman/synthserver/master/pics/closeup.png)

(please note, in this picture the GAIN pin is connected incorrectly, it should be on Vin) 

## Firmware

Setup esp-idf: http://esp-idf.readthedocs.io/en/latest/get-started/

Make sure to add an auth.h in the main/ folder with 
```
#define WIFI_SSID "your ssid"
#define WIFI_PASS "your password"
```

Just run `idf.py -p /dev/YOUR_SERIAL_TTY flash` to build and flash to the board after setup.

## Using it

Send commands via UDP in ASCII delimited by a character, like

```
v0w4f440.0a0.5
```

Where
```
a = amplitude, float 0-1 summed over all voices. default 0
c = client, uint, 0-255 indicating a single client, 256-510 indicating (client_id % (x-255) == 0) for groups, default all clients
f = frequency, float 0-22050. default 0
n = midinote, uint, 0-127 (note that this will also set f). default 0
p = patch, uint, 0-1000, choose a preloaded DX7 patch number for FM waveforms. default 0
s = sync, int64, same as time but used alone to do an enumeration / sync, see tones.py, also uses i for sync_index
t = time, int64: ms since some fixed start point on your host. you should always give this if you can
v = voice, uint, 0 to 9. default: 0
w = waveform, uint, 0,1,2,3,4,5,6 [SINE, SQUARE, SAW, TRIANGLE, NOISE, FM, OFF]. default: 0/SINE
```

Commands are cumulative, state is held per voice. If voice is not given it's assumed to be 0. 

Example:

```
f440a0.1t4500
a0.5t4600
w1t4600
v1w5n50a0.2t5500
v10.4t7000
w2t7000
```

Will set voice 0 (default) to a sine wave (default) at 440Hz amplitude 0.1, then set amplitude of voice 0 to 0.5, then change the waveform to a square but keep everything else the same. Then set voice 1 to an FM synth playing midi note 50 at amplitude 0.2. Then set voice 1's amplitude to 0.4. Then change voice 0 again to a saw wave.

## Addressing individual synthesizers

By default, a UDP multicast message reaches all booted synthesizers at once (see below for timing info.) If you want to address a single synth, or half or a quarter of them, use the `client` parameter. Keeping `client` off sends a message to all synths. Setting `client` to a number between 0 and 255 will only reach the synthesizer with that client ID. Client ID is simply the last octet of its IPV4 address. To learn which client IDs are online, use the `sync` command and see `tones.py`'s implementation.

Setting `client` to a number greater than 255 allows you to address groups. For example, a `client` of 257 performs the following check on each booted synthesizer: `my_client_id % (client-255) == 0`. This would only address every other synthesizer. A `client` of 259 would address every fourth synthesizer, and so on.

## Timing & latency & sync

WiFi, UDP multicast, distance, microcontrollers with bad antennas: all of these are in the way of doing anything close to "real time" control from your host. A message you send from a laptop will arrive between 5ms and 200ms to the connected speakers, and it'll change each time. That's definitely noticeable. We mitigate this by setting a global latency, right now 500ms, and by allowing any host to send along a `time` parameter of when the host expects the sound to play. `time` can be anything, but I'd suggest using the number of milliseconds since the "alles epoch", e.g.

```
def alles_ms():
    return int((datetime.datetime.utcnow() - datetime.datetime(2020, 2, 1)).total_seconds() * 1000)
```

The first time you send a message the synth uses this to figure out the delta between its time and your expected time. (If you never send a time parameter, you're at the mercy of both fixed latency and jitter.) Further messages will be accurate message-to-message, but with the fixed latency. 

If you want to update this delta (to correct for drift over time or clock base changes) use the `sync` command. See `tones.py`'s implementation, but sending an `sTIMEiINDEX` message (preferably regularly, e.g. 10 messages once every 100ms) will update the delta and also trigger a response back from each on-line synthesizer. The response looks like `_s65201i4c248`, where s is the time on the client, i is the index it is responding to, and c is the client id. This lets you build a map of not only each booted synthesizer, but also be able to figure the round-trip latency for each one along with the reliability. This is helpful when your synths are spread far apart in space and each may have unique latencies. e.g. here we see we have two synths booted on a busy WiFi network.

```
>>> tones.sync(count=10)
{
	248: {'avg_rtt': 319.14285714285717, 'reliability': 0.7}, 
	26: {'avg_rtt': 323.5, 'reliability': 0.8}
}
```

And here's a response with 4 synths on a local wired dedicated router:

```
>>> tones.sync()
{
	3: {'avg_rtt': 4.0, 'reliability': 1.0},
	4: {'avg_rtt': 4.6, 'reliability': 1.0},
	5: {'avg_rtt': 4.2, 'reliability': 1.0},
	6: {'avg_rtt': 5.2, 'reliability': 1.0}
}
```

## WiFi & reliability 

UDP multicast is naturally 'lossy' -- there is no guarantee that a message will be received by a synth. Depending on a lot of factors, but most especially your wireless router and the presence of other devices, that reliability can go between 70% and 99%. In my home network, a many-client Google WiFi mesh, my average round trip latencies are in the high 200 ms range, and my reliability is in the 75% range. On a direct wired Netgear Nighthawk AC2300 with only synthesizers as clients, latencies are well under 50ms and reliability is close to 100%. For performance purposes, I highly suggest using a dedicated wireless router instead of an existing WiFi network. You'll want to be able to turn off many "quality of service" features (these prioritize a randomly chosen synth and will make sync hard to work with), and you'll want to in the best case only have synthesizers as direct WiFi clients. Using a standalone router also helps with WiFi authentication setup -- by default the login & password for WiFi is `alles` and `sellasella` on all synthesizers. If you were using your own network you'll have to recompile with your own authentication, which gets tiresome with many synths. 

An easy way to do this is to set up a dedicated router but not wire any internet into it. Connect your laptop or host machine to the router over a wired connection (via a USB-ethernet adapter if you need one) from the router, but keep your laptop's wifi or other internet network active. In your controlling software, you simply set the source network address to send and receive multicast packets from. See `tones.py` for more details on this. This will keep your host machine on its normal network but allow you to control the synths from a second interface.

If you're in a place where you can't control your network, you can mitigate reliability by simply sending messages N times (2-4). Since individual messages are stateless and can have target timestamps, sending multiple duplicate messages do not have any averse effect on the synths.


## Clients

Simple Python example:

```
import socket, struct
multicast_group = ('232.10.11.12', 3333)
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, struct.pack('b', 1))

def tone(voice=0, type=0, amp=0.1, freq=0):
    sock.sendto("v%dw%da%ff%f" % (voice, type, amp, freq), multicast_group)

def c_major(octave=2,vol=0.2):
    tone(voice=0,freq=220.5*octave,amp=vol/3.0)
    tone(voice=1,freq=138.5*octave,amp=vol/3.0)
    tone(voice=2,freq=164.5*octave,amp=vol/3.0)

```

See `tones.py` for a better example.

You can also use it in Max or similar software (note you have to wrap string commands in quotes in Max, as otherwise it'll assume it's an OSC message.)

![Max](https://raw.githubusercontent.com/bwhitman/synthserver/master/pics/max.png)

## THANK YOU TO

* douglas repetto
* dan ellis
* [MSFA](https://github.com/google/music-synthesizer-for-android) for FM impl

## TODO

* ~~remove distortion at higher amplitudes for mixed sine waves~~
* ~~FM~~
* envelopes / note on/offs
* wifi hotspot mode for in-field setup (tbh think it's better to use a dedicated router)
* ~~broadcast UDP for multiples~~
* ~~dropped packets~~ (although more work can be done here)
* ~~sync and enumerate across multiple devices~~
* ~~addresses / communicate to one or groups, like "play this on half / one-quarter / all"~~
* ~~do what i can about timing / jitter - sync time? timed messages?~~
* case / battery setup
* overloading the volume (I think only on FM) crashes
* UDP message clicks 



