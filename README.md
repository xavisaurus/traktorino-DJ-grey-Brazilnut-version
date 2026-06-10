Traktorino VirtualDJ Firmware Changes

This document summarises the firmware changes made to Traktorino.ino compared with the original upstream Traktorino sketch.

The goal of these changes is to make the original Arduino Uno-based Traktorino firmware work more reliably with VirtualDJ, especially for MIDI feedback, button LEDs, and VU meter behaviour.

Scope of Changes

The firmware changes are intentionally limited to MIDI feedback reliability and VU meter handling.

The following areas are not implemented in firmware:

Shift-layer DJ workflow
Smart Fader behaviour
Cue play / stutter restart behaviour
Loop size control
Key lock, unload, pitch reset and other secondary functions

These behaviours are handled in the VirtualDJ mapper XML, not in the Arduino firmware.

1. MIDI Instance Enabled
Original Behaviour

The original sketch had the MIDI instance creation line commented out:

//MIDI_CREATE_DEFAULT_INSTANCE();

This caused compilation issues when the sketch referenced the MIDI object.

Updated Behaviour

The MIDI instance is now enabled:

MIDI_CREATE_DEFAULT_INSTANCE();
Reason

This is required for calls such as:

MIDI.read();
MIDI.sendNoteOn();
MIDI.sendControlChange();

Without this, the sketch cannot compile correctly with the FortySevenEffects MIDI library.

2. MIDI Input Initialisation Added
Original Behaviour

The original firmware used MIDI output and MIDI input callbacks, but did not explicitly initialise MIDI input for feedback handling in a VirtualDJ-oriented setup.

Updated Behaviour

MIDI input is initialised in setup():

MIDI.begin(MIDI_CHANNEL_OMNI);
Reason

VirtualDJ sends MIDI feedback back to the Arduino for:

Button LEDs
VU meter LEDs
Other future output feedback

Using MIDI_CHANNEL_OMNI allows the firmware to receive feedback regardless of the MIDI channel used by the software.

3. MIDI Read Loop Improved
Original Behaviour

The original loop processed only one MIDI message per loop cycle:

MIDI.read();
Updated Behaviour

The firmware now drains all pending MIDI messages each loop:

while (MIDI.read()) {
  // Drain pending MIDI messages for more responsive LED/VU feedback.
}
Reason

VU meter feedback can generate frequent MIDI Control Change messages. Processing only one message per loop can cause lag or stale LED values.

This change improves responsiveness for:

VU meter updates
LED feedback
Rapid MIDI output from VirtualDJ
4. Note On Velocity 0 Treated as Note Off
Original Behaviour

The original handleNoteOn() function treated any Note On message as an LED-on event, regardless of velocity.

This means the following message could still turn a LED on:

Note On, velocity 0
Updated Behaviour

The firmware now treats Note On with velocity 0 as Note Off:

if (value == 0) {
  handleNoteOff(channel, number, value);
  return;
}
Reason

Many MIDI applications use this convention:

Note On velocity 127 = LED on
Note On velocity 0   = LED off

This change improves compatibility with VirtualDJ and other MIDI software that uses velocity-zero Note On messages for LED-off events.

5. VU Meter State Tracking Split by Side
Original Behaviour

The original firmware used a single shared variable to track the last VU CC value:

int ccLastValue = 0;

This single value was shared between left and right VU meters.

Problem

If both VU meters received the same value, the second update could be ignored because the firmware believed the value had not changed.

Example:

Left VU receives value 4
Right VU receives value 4

The right side may not update because the shared last value was already 4.

Updated Behaviour

Separate state tracking is now used:

byte vuLastValueL = 255;
byte vuLastValueR = 255;
Reason

Left and right VU meters must be tracked independently.

This avoids one side blocking or suppressing updates to the other.

6. VU Meter Timeout Added
Original Behaviour

If VirtualDJ stopped sending VU meter values when playback stopped, the Arduino could continue displaying the last received VU level.

This caused the VU LEDs to remain lit after stopping a track.

Updated Behaviour

The firmware now tracks when each VU meter last received an update:

unsigned long vuLastUpdateL = 0;
unsigned long vuLastUpdateR = 0;
const unsigned long vuTimeoutMs = 220;

A new timeout function clears stale VU meters:

void clearVuIfStale() {
  unsigned long now = millis();

  if ((now - vuLastUpdateL > vuTimeoutMs) && vuLastValueL != 0) {
    setVuMeter(VuL, 0);
    vuLastValueL = 0;
  }

  if ((now - vuLastUpdateR > vuTimeoutMs) && vuLastValueR != 0) {
    setVuMeter(VuR, 0);
    vuLastValueR = 0;
  }
}

This function is called from loop().

Reason

VU meters are stream-based. If the MIDI stream stops without sending a final zero value, the hardware will keep showing the last known level.

The timeout makes the firmware self-clearing.

Tuning

The timeout can be adjusted here:

const unsigned long vuTimeoutMs = 220;

Suggested values:

Value	Behaviour
150	Faster clearing, may feel slightly abrupt
220	Balanced default
300	Smoother, slower clearing
7. VU Meter Drawing Rewritten
Original Behaviour

The original VU handling used repeated switch blocks and directly controlled individual LED states.

Updated Behaviour

A helper function now redraws the full VU meter:

void setVuMeter(unsigned int vuPins[7], byte level)

This function:

Accepts the VU level
Normalises it to the expected range
Clears all seven LEDs
Redraws the correct number of LEDs
Applies colour logic for green, yellow and red LEDs
LED Colour Logic
Level	Result
0	All VU LEDs off
1-5	Green LEDs
6	Green LEDs + yellow
7	Green LEDs + yellow + red
Reason

Clearing and redrawing the full VU state is more reliable than updating individual LEDs incrementally.

It prevents old LEDs from remaining lit after the level drops.

8. VU Input Range Expanded
Original Behaviour

The original firmware expected VU values in the range:

0 to 7
Updated Behaviour

The firmware now accepts both:

0 to 7

and standard MIDI-style values:

0 to 127

If a value above 7 is received, it is scaled down:

if (level > 7) {
  level = map(level, 0, 127, 0, 7);
}

The result is constrained:

level = constrain(level, 0, 7);
Reason

VirtualDJ may send controller feedback in a normal MIDI range of 0-127, while the Traktorino VU hardware has only seven LED levels.

This makes the firmware tolerant of both compact and standard MIDI ranges.

9. Updated Control Change Handling
Original Behaviour

The original handleControlChange() logic handled VU values with shared state and fixed assumptions.

Updated Behaviour

The updated logic handles:

CC 12 = Left VU
CC 13 = Right VU

Each side is tracked separately:

if (number == 12) {
  vuLastUpdateL = millis();

  if (level != vuLastValueL) {
    setVuMeter(VuL, level);
    vuLastValueL = level;
  }
  return;
}

if (number == 13) {
  vuLastUpdateR = millis();

  if (level != vuLastValueR) {
    setVuMeter(VuR, level);
    vuLastValueR = level;
  }
  return;
}
Reason

This improves stability and makes the VU output more predictable in VirtualDJ.

10. Shift Layer Kept Out of Firmware
Design Decision

No Shift-layer workflow logic has been added to the Arduino firmware.

The Shift button remains a normal MIDI button:

Shift = MIDI note 0x2A
Reason

The firmware should remain responsible for:

Reading physical controls
Sending MIDI input to the host
Receiving LED feedback
Driving the LED hardware

The DJ workflow should remain in VirtualDJ mapping, including:

Shift + Crossfader = Smart Fader while held
Shift + Play = Cue Play / Stutter Restart
Shift + Cue = Set Cue
Shift + Sync = Pitch Reset
Shift + PFL = Key Lock
Shift + Load = Unload Deck
Shift + Browser Encoder = Loop Size

This keeps the firmware reusable and avoids reflashing the Arduino every time the DJ workflow changes.

11. Smart Fader Not Implemented in Firmware
Design Decision

Smart Fader behaviour is not implemented in the .ino file.

Reason

Smart Fader is a software-level function in VirtualDJ. The Arduino only sends crossfader movement and Shift button state.

The VirtualDJ mapper handles this using logic equivalent to:

Shift held = smart_fader on
Shift released = smart_fader off

This is more flexible and avoids hardcoding software-specific behaviour into the controller firmware.

12. Backwards Compatibility Notes

The firmware remains compatible with the original Traktorino hardware design, assuming:

Arduino Uno or compatible AVR board
Original button note layout
Original CC layout for sliders and VU meters
ShiftPWM LED wiring remains unchanged

The main compatibility change is improved MIDI feedback handling for VirtualDJ.

13. Known Notes and Considerations
Button LED Feedback

The firmware listens for MIDI note feedback on the same note numbers as the physical buttons.

Examples:

LED	MIDI Note	Hex
Left Play LED	38	0x26
Right Play LED	46	0x2E
Left Cue LED	39	0x27
Right Cue LED	45	0x2D
Left Sync LED	40	0x28
Right Sync LED	44	0x2C
VU Feedback
VU Meter	CC	Hex
Left VU	12	0x0C
Right VU	13	0x0D

The firmware accepts either 0-7 or 0-127 values.

VirtualDJ Device Definition

The recommended VirtualDJ device definition keeps the original internal device identity:

name="SIMPLE_MIDI_9025_29908"
description="Traktorino"

This avoids VirtualDJ detecting the controller as a new or blank device every time it is connected.

Summary

The firmware changes are focused on making the original Traktorino sketch more reliable for VirtualDJ usage.

Main Improvements
Fixes MIDI object creation
Improves MIDI feedback reception
Handles Note On velocity 0 as LED off
Makes VU meters more responsive
Prevents VU meters from staying lit after playback stops
Supports both 0-7 and 0-127 VU values
Keeps Shift-layer DJ workflow in VirtualDJ mapping instead of firmware
Not Changed
Hardware pin layout
Button note layout
Slider CC layout
Core Traktorino control scanning
ShiftPWM LED hardware structure
DJ workflow logic
