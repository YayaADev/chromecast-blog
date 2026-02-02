---
layout: default
title: Building a Chromecast Audio Clone with an ESP32
---

# Building a Chromecast Audio Clone with an ESP32

I wanted to stream audio to my soundbar over WiFi. Google made a product for this (Chromecast Audio) but they killed it in 2019. So I built my own with an ESP32 my coworker gave me one day.

My soundbar has three inputs: HDMI eARC (occupied by the TV), Bluetooth (too much latency/compression), and TOSLINK. So I went with TOSLINK. It turns out, all TOSLINK really does is flash light pulses. And an ESP32 flashing an LED is like the "Hello World" of embedded systems.

## System Overview

![System Overview](/chromecast-blog/assets/images/esp32-system-desing.png)

## Wiring & Pinout

| **Signal**     | **ESP32 Pin** | **Connection**        |
| -------------- | ------------- | --------------------- |
| **I2S Data**   | GPIO 22       | Anode (+) of Red LED  |
| **IR Blaster** | GPIO 23       | IR LED (via resistor) |
| **Ground**     | GND           | LED Cathodes (-)      |

![ESP32 board with LED connections](/chromecast-blog/assets/images/ESP_Board.jpg)

To stream the audio, I used **ffmpeg** to push raw PCM data over UDP.

```bash
ffmpeg -re -i file.mp3 \
  -acodec pcm_s16le -ar 44100 -ac 2 \
  -f s16le udp://esp32-spdif.local:1234
```
> This command converts the mp3 into raw PCM data at 44.1kHz stereo and streams it over UDP to the ESP32. The `-re` flag makes ffmpeg send the data in real-time instead of as fast as possible, which would overwhelm the buffer.

## Receiving Audio over WiFi

The ESP32 continuously listens for incoming audio packets over WiFi. The issue with WiFi is that it isn't perfectly steady. Packets can arrive in bursts, get delayed, or bunch up because the router is busy doing other things. If I just played audio the instant it arrived, every tiny network hiccup would cause an audible glitch. Instead I queue the audio up in a ring buffer. A chunk of memory that smooths the process of incoming audio to output audio. It's like a doubly linked list and processes the audio as it goes, once full it overwrites the old audio and keeps going.

Full code can be found [AudioTransmitter](https://github.com/YayaADev/AudioTransmitter/blob/master/audio-transmitter.ino)
### Ring buffer

I set the ring buffer size to 256KB. At 44.1kHz stereo 16-bit (I think S/PDIF can do 96kHz but my soundbar maxes out at [48kHz](https://videoandaudiocenter.com/sony-hts2000-3-1ch-dolby-atmos-soundbar/#:~:text=HDMI%20eARC-,HDMI%20eARC%20connection%20and%20power%20on.,in%20favor%20of%20recycled%20materials.) sI just went with the default CD number), one second of audio is 176,400 bytes (44,100 samples × 2 channels × 2 bytes), so 256KB gives roughly ~1.49 seconds of buffer. Probably overkill, could be much lower. 

I also set a minimum buffer threshold of 4KB before playback starts. I didn't have this at first and would sometimes hear a click/pop sound which is a [buffer underrun](https://en.wikipedia.org/wiki/Buffer_underrun) i think. When the buffer empties mid-playback there's a sudden jump in the waveform when new audio arrives. The discontinuity is the click.

Each loop iteration pulls 1KB chunks from the buffer to send to S/PDIF. Smaller chunks are more responsive but cost more CPU overhead. Larger chunks will bring in more latency from recieving and audio, especially on first play. 1KB felt like a reasonable middle ground.
### UDP

Using UDP instead of TCP since this is real time audio streaming. It would be a poor experience if on every dropped packet I let TCP do its guaranteed delivery thing. Audio pauses, buffer drains, then resumes with a larger glitch. With UDP if a packet goes missing you wouldn't even hear it. The library [AsyncUDP](https://github.com/me-no-dev/ESPAsyncUDP.git) handles this for me

### mDNS

Using mDNS was a handy way for me to have a consistent name for streaming audio. The ESP gets its IP address from my router via **DHCP**. The router may assign a different IP after a reboot, network change, or something. I'd have to keep figuring out the new IP and change the ffmpeg command. Instead I have a consistent URL to forward audio to `udp://esp32-spdif.local:1234`

---

## Understanding S/PDIF and TOSLINK

This part took most of my time as I was over complicating it and didnt realize its actually a lot simpler than I initially thought.

### S/PDIF

S/PDIF (Sony/Philips Digital Interface) is a standard for transmitting digital audio between devices. It was built in the 1980s to overcome the signal degradation that came with analog (at least that's what I read, I grew up in an HDMI era). Instead of analog waveforms, S/PDIF sends audio as raw binary, just 1s and 0s.

Digital communication usually needs a separate clock wire so devices agree on timing ([HDMI](https://docs.altera.com/r/docs/683798/23.4/hdmi-ip-user-guide/hdmi-intel-fpga-ip-quick-reference)  does this). But S/PDIF was designed as a single wire for simpler consumer setups. The All In One audio wire.  To pack both timing and data into one wire, it uses biphase mark encoding: every bit flips the signal at least once so the receiver can track timing, and 1s get an extra flip in the middle to distinguish them from 0s.

### TOSLINK

TOSLINK is just S/PDIF transmitted as light pulses through a fiber optic cable instead of electrical signals on a wire. It's the same protocol, a TOSLINK transmitter converts the electrical S/PDIF signal to light (LED on/off) and receivers convert it back.

#### How the SPDIFOutput Library Works

The [AudioTools (by Phil Schatzmann)](https://github.com/pschatzmann/arduino-audio-tools.git) library handles the work of converting raw audio data into a S/PDIF signal by repurposing the I2S in the ESP32.

The I2S is a hardware piece with three wires BCLK (Bit Clock), LRCLK (Left/right clock), DATA (the actual audio bits) and is usually used with a DAC chip. Since S/PDIF sends it all with one wire it only uses the `DATA` wire. The process is essentially this

1. Takes the raw PCM audio bytes.
2. Adds S/PDIF formatting (timing info + audio data)
3. Encodes it so the receiver can extract both data and timing from one signal
4. Sends it to the I2S
5. The I2S outputs the signal on GPIO 22

---

## Outputting Audio with an LED

At 44.1kHz, that $0.20 LED $ is flashing around 2.8 million times per second! After the audio is converted its being sent on the GPIO pin but its still an electrical signal and TOSLINK needs light. Typically you'd use a TOSLINK transmitter to convert the two like a [TORX1950A(F)](https://www.mouser.com/ProductDetail/Toshiba/TORX1950AF?qs=gev7jUp%252BQ%252BhPDbv%252Bwuk0PA%3D%3D). But thats boring and I wanted a DIY solution. 

It turns out that an LED is really all thats technically need to convert the electric audio signal to light. The actually conversion process is called [electroluminescence](https://en.wikipedia.org/wiki/Electroluminescence) (this is now physics realm). The LED however is important [TOSLINK transmitters operate at a nominal optical wavelength of 650 nm](https://en.wikipedia.org/wiki/TOSLINK). I got a 20 pack for about $4 with these specs.

**Color:** Red (~630nm wavelength)
- **LED Size:** 3mm (T1 package)
- **Millicandela (Brightness):** 5000mcd
- **Forward Voltage:** 2.1V DC (typical)
- **Forward Current:** 20mA
- **Viewing Angle:** 60°
- **Lens Type:** Diffused, colored

The wavelength here is what matters most as you need something as close to 650 as possible. A smaller LED with a viewing angle like 60 is also likely to work better as the light isnt being scattered around and more concentrated (think of your phone flashlight vs a cat laser toy). Paired that with a 68Ω resistor to run it near 20mA

```
R = (V_supply - V_forward) / I_forward R = (3.3V - 2.1V) / 0.020A = 60Ω
68 is close enough
```

The final piece was actually figuring out how do you connect the end of a TOSLINK cable to an LED? The LED is not a plug so it wont just fit inside (unless you got a [COB](https://lumileds.com/technology/led-technology/understanding-cob-leds/) LED and can mount the LED die). The easiest solution was to get a coupler that can hold the two in place, and there was a [STL file available on thingiverse](https://www.thingiverse.com/thing:5705718) that does exactly that!

## Bonus: Auto Wake-up via IR Blaster

`[This was a separate problem I ran into after getting the audio working.]`

I normally use my soundbar with the TV. An HDMI eARC cable connects the two. When the TV turns on or off soundbar automatically turns on or off with it. But with a TOSLINK cable to an LED, the signal to turn on and off is not passed. The other way to turn on a soundbar is with the provided remote which sends an IR blast signal to the soundbar. I took the idea of using an IR blaster and decided to create my own IR blaster on the ESP to control the soundbar too.

An issue I ran into was actually figuring out what the codes were. The remote bundled with the soundbar is a [RMT-AH513U](https://www.sony.com/electronics/support/sound-bars-home-theater-systems-sound-bars-surround-speakers/ht-s2000). 

Trying to find the codes online, I discovered there is a database that hosts all remote IR codes called [LIRC](https://lirc-remotes.sourceforge.net/) (I also found out about [Global Cache](https://irdb.globalcache.com/Home/Database) which looks to be a bigger repository of codes, but its mostly paid and requires an account for viewing and thats annoying). Unfortunately the data here didn't have my specific remote available as its a bit old. 

I needed to manually figure out the codes myself with an IR receiver and send the signals to it. Luckily I had one with a built-in IR Receiver. LIRC provides a CLI tool called [irrecord](https://sourceforge.net/projects/lirc/) to run that can capture the signals and generates a config file on the information captured. I tried to test every single button to know it just in case but sometimes during a session the CLI tool would crash and i'd have to start all over again. 

I didnt know why, maybe the receiver i have isnt great, or there was interference ? Couldn't tell but after a few times it became too time consuming so i just tested with the buttons i care about (power, input, volume) and exited. [sony-ht-s2000.lircd.conf](https://gist.github.com/YayaADev/d393afb27e7f5e17e858e7b596ce9311)

With the codes the second part was building the IR blaster on the ESP. This was simpler as all i needed was a IR LED and a resistor for the LED. I used a the package [IRremoteESP8266] as it supports sending IR signals.

```
rsend.sendSony38(0x540C, 15, 2);
// 0x540C - the code from the file
// 15 - this is a 15 hex code
// 2 - This sends the code twice. Sony needs multiple signals to turn on
```

## Results

I have two clips showing it in action.

**Audio streaming over WiFi to the soundbar via TOSLINK:**

<div class="video-container">
  <video controls>
    <source src="/chromecast-blog/assets/videos/VidNoIR.mp4" type="video/mp4">
    Your browser doesn't support the video tag.
  </video>
</div>

This clip shows the audio being transmitted through the LED without the coupler. The lights are dimmed to put more focus on the LED and really capture when the data is being transmitted. I ran the ffmpeg command before recording to keep the LED lit while holding the cable.

**IR blaster automatically turning on the soundbar:**

<div class="video-container">
  <video controls>
    <source src="/chromecast-blog/assets/videos/VidWithIR.mp4" type="video/mp4">
    Your browser doesn't support the video tag.
  </video>
</div>

This shows everything working together. The ESP32 is positioned directly in front of the soundbar's IR receiver (the IR LED signal is pretty weak and needs to be close). The TOSLINK cable is held with the 3D-printed coupler to the LED. You can see the soundbar turn on and hear the audio start playing after running ffmpeg. There's some latency between running the command and audio playing, but that's mainly the soundbar wake-up time.

[Audio used for testing](https://pixabay.com/music/dubstep-dark-cyberpunk-i-free-background-music-i-free-music-lab-release-469493/)

