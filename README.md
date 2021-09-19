# Screenshare with audio on Discord with Linux
This is a WIP

## Prologue
Screensharing on Discord has been a huge pain for non Windows users since desktop audio would not be captured and until recently there was no option to even pick one screen if you had multiple. Screensharing with desktop audio was recently fixed on Mac OS on the official client only through a propriatary hack since getting desktop audio on Mac isn't easy and you need something like soundflower to interact with the kernel and electron/chromium don't have such functionality.<br>
A hack many people do is to mix their microphone with their desktop audio to stream it that way, this is not proper because all users on the call are forced to listen to your desktop audio regardless of whether they are watching your stream or not (huge issue when many people are streaming), they cannot adjust your stream and microphone volume individually, mic channels overlap on discord when many people are talking, bitrate is low, the sound is mono and effects are applied. Some of these issues may appear in our methods too but its possible to fix them using JavaScript which is why this repo exists in the first place.

## Enter Chromium
As you probably know Discord's client isn't a native app, its actually a web browser since it uses electron that is basically a modified version of Chromium. You can use Chromium or any Chromium-Based browser like Brave and Vivaldi to login on Discord and you can even Screenshare but desktop audio will only work on Windows and Linux with Chromium OS Audio Server (so only Chrome/Chromium OS). Linux and Mac users can only share the audio of a tab.<br>
As discussed in the prologue its hard to get desktop audio on Mac OS because a signed kernel extension is needed so it must be hard on Linux too right? The answer is no, in fact any app that can capture an input can also capture desktop audio, that's because on PulseAudio (includes PipeWire) there exist some virtual input devices called Monitors. Monitors basically replicate the audio of an output (like your speakers or headphones) and make it an input. The sad part of the story is that Chromium developers deliberately chose to blacklist Monitor devices on Chromium (see [here](https://bugs.chromium.org/p/chromium/issues/detail?id=931749) and [here](https://chromium.googlesource.com/chromium/src/+/4519c32f528e079f25cb2afc594ecf625f943782)). But if you are familiar with Linux's audio server you'd know that even if Chromium refuses to capture a Monitor, if it captures any other input devices like a microphone we can then manipulate what Chromium captures through pulseaudio assistant programms like pavucontrol or a patchbay like helvum and qjackctl (we run it using `pw-jack qjakckctl` on pipewire)

## The Plan
It's a pretty simple one and emanates from the paragraph above. Basically, since Chromium cannot capture a monitor we'll capture a microphone, merge it with the Screenshare stream using JavaScript and then connect the audio of the apps we want using a patchbay like helvum or qjackctl.

## The Problems
As mentioned in the prologue there are still some issues to be resolved.
1. Screenshare is only 720p 30fps. This can probably be fixed by forcing frameRate, width and heigh Video constraints
2. Audio on both the microphone is stream and the desktop stream is in mono. Forcing the channelCount: 2 constraint solely doesn't fix it. (Check [that](https://bugs.chromium.org/p/chromium/issues/detail?id=387737))
3. This doesn't work on the Discord electron client. Since electron uses getUserMedia to share your screen instead of getDisplayMedia we need seperate code to achieve the same result.

