# Overall Goal

Bruce Hall's implementation of the morse code tutor (https://github.com/bhall66/morse-tutor/blob/master/) is a wonderful tool for learning morse code, but I found that the sound 
quality was harsh when I used headphones.

I'm not a C++ programmer, but I asked ChatGPT to help me try to refactor the code to improve the sound quality. The goal of the project was to replace the original square wave 
tone (used for Morse code beeps) with a smoother sine wave, which would produce a softer, less percussive sound.

This project is a fork of Bruce Hall's repository.

# How We're Achieving It

1. We had to bypass the normal audio for the tutor by adding jumpers directly off the ESP32 board to use the built-in DAC (digital-to-audio-converters) on 
the board. Here's what ChatGPT has us do:

    * Find GPIO25 on the ESP32 Board. GPIO25 will be labeled as 25 on the silkscreen or board diagram.
    * Wire GPIO25 to the audio output. Temporarily, run a jumper wire from GPIO25 to the input of your audio amplifier or the 3.5 mm jack’s input line. Later, with the help of a friend, we made an inline filter using a 10 µF capacitor in series between GPIO25 and the audio input to block DC (important with DAC).
    * Set up the ground jumper. We made sure the GND of your ESP32 was connected to the GND of your amplifier.

2. Use the ESP32’s Built-in DAC. The ESP32 has two 8-bit Digital-to-Analog Converters (DACs) on pins 25 and 26. We use `dacWrite(pin, value)` to output a voltage corresponding to a value between 0–255.
So:

    * `dacWrite(25, 128)` = ~midpoint voltage (silence)
    * `dacWrite(25, 255)` = max voltage
    * `dacWrite(25, 0)` = min voltage

  This lets us output a waveform by writing a sequence of values over time.

3. Simulate a sine wave. We use a lookup table `(sineTable[])` to represent a full cycle of a sine wave. The values go from 128 (center), 
to 255 (peak), back to 128, then to 0 (valley), then back to 128. This table is indexed by a timer interrupt `(onTimer()`), 
which sends one value at a time to the DAC, producing a continuous voltage curve. This emulates a sine wave—not with analog voltage 
generated from hardware oscillators, but by stepping through precomputed values at a fixed sample rate. 
This is digital synthesis of an analog tone, and it sounds clean compared to a square wave.

4. Apply an Envelope (Amplitude Shaping). The sine wave alone can still "click" when it starts and stops abruptly. So we apply a volume envelope that fades 
in at the start of a tone fades out at the end. This is done using another lookup table `(envelopeTable[])` which multiplies the sine wave’s 
amplitude gradually from 0 to 1 and then back to 0. This reduces the "attack" and "release" harshness—especially important for Morse tones 
where each dit and dah is short.

# Summary of What the DAC Routine Does

The DAC simply outputs direct voltage levels, but by rapidly changing those values (according to a sine function), we can:

* Emulate smooth analog waveforms
* Shape them with an envelope function
* Replace old square-wave pulses with audio-quality tones

# Summary of software changes

Here’s a summary of the functions, definitions, and code regions changed in this project:

* DAC sine wave audio
* Envelope smoothing
* Rewiring pitch code to accept DAC output
* SD card filename and file handling

## DAC Sine Wave Output

Here are some changes for the sine wave emulation:

### New or modified variables

    ```
	#define DAC_PIN 25
	#define SINE_TABLE_SIZE 256
	uint8_t sineTable[SINE_TABLE_SIZE];
	```

### New or Modified Functions

* `initSineTable()`—initializes sine waveform table
* `onTimer()`—ISR generating sine wave output via DAC
* `hitTone()` and `missTone()`

## Envelope Smoothing (Cosine Envelope)

We made several changes to add the smoothing:

### New or modified variables

* `#define ENVELOPE_MAX SINE_TABLE_SIZE`
* `uint8_t envelopeTable[ENVELOPE_MAX + 1];`
* `volatile bool envelopeActive = false;`

### New or Modified Functions

* `initEnvelopeTable()`—fills envelopeTable with a cosine-shaped ramp
* `onTimer()`—extended to apply envelope shaping during DAC output
* `keyDown()` and `keyUp()`—may now set envelopeActive = true/false

## Reconnect Pitch Menu to DAC frequency

Previously, the DAC tone was driven by a fixed value of pitch from `setup()`. This meant that the pitch adjustment
menu didn't actually change the pitch anymore.

To make the menu-adjustable pitch take effect again, we added this function to reconfigure the timer on pitch change:

    ```
	void updateToneFrequency() {
	  int freq = pitch * SINE_TABLE_SIZE;  // pitch is WPM-ish, table is samples/cycle
	  timerAlarmWrite(timer, 1000000 / freq, true);  // update interval for new frequency
	}
	```
	
Then ChatGPT had us call `updateToneFrequency();` in `setPitch()`.

This successfully enabled the pitch menu to play and accept pitch changes from DAC.

## SD Card Fixes

The SD card routine was doing two incorrect things prior to the sine wave refactoring:

* It was truncating the first character of each filename.
* It was not listing files on separate lines.

After the refactoring to smooth the sound, the SD card files wouldn't play at all.

I learned that the files need to follow the 8.3 pattern; so I shortened the names. This seemed to 
allow the files to be listed on separate lines.

Here are some other changes that ChatGPT made to get the SD card code to work again:

### Modified Definitions:


* `#define FNAMESIZE 15`   // or whatever your filename buffer size is
* `#define MAXFILES  10`   // limit of how many files are listed

### Modified Functions:

* `getFileList(char list[][FNAMESIZE])`
* Replaced unsafe strcpy() with safe strncpy()
* Added `if (name[0] == '/') name++;` to handle ESP32 file paths
* Added serial debug output
* `sendFile(char* filename)` now uses strncat() safely
