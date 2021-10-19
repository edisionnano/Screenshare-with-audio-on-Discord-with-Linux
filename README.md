# Screenshare with audio on Discord with Linux
This is a WIP

## Prologue
Screensharing on Discord has been a huge pain for non Windows users since desktop audio would not be captured and until recently there was no option to even pick one screen if you had multiple. Screensharing with desktop audio was recently fixed on Mac OS on the official client only through a proprietary hack since getting desktop audio on Mac isn't easy and you need something like Soundflower to interact with the kernel and electron/chromium don't have such functionality.<br>
A hack many people do is to mix their microphone with their desktop audio to stream it that way, this is not proper because all users on the call are forced to listen to your desktop audio regardless of whether they are watching your stream or not (a huge issue when many people are streaming), they cannot adjust your stream and microphone volume individually, mic channels overlap on discord when many people are talking, bitrate is low, the sound is mono and effects are applied. Some of these issues may appear in our methods too but it's possible to fix them using JavaScript which is why this repo exists in the first place.

## Enter Chromium
As you probably know Discord's client isn't a native app, it's a web browser since it uses electron that is a modified version of Chromium. You can use Chromium or any Chromium-Based browser like Brave and Vivaldi to log in on Discord and you can even Screenshare but desktop audio will only work on Windows and Linux with Chromium OS Audio Server (so only Chrome/Chromium OS). Linux and Mac users can only share the audio of a tab.<br>
As discussed in the prologue it's hard to get desktop audio on Mac OS because a signed kernel extension is needed so it must be hard on Linux too right? The answer is no, in fact, any app that can capture an input can also capture desktop audio, that's because on PulseAudio (includes PipeWire) there exist some virtual input devices called Monitors. Monitors replicate the audio of an output (like your speakers or headphones) and make it an input. The sad part of the story is that Chromium developers deliberately chose to refuse to list or capture Monitor devices on Chromium (see [here](https://bugs.chromium.org/p/chromium/issues/detail?id=931749) and [here](https://chromium.googlesource.com/chromium/src/+/4519c32f528e079f25cb2afc594ecf625f943782)). But if you are familiar with Linux's audio server you'd know that even if Chromium refuses to capture a Monitor, if it captures any other input devices like a microphone we can then manipulate what Chromium captures through pulseaudio assistant programms like pavucontrol or a patchbay like helvum and qjackctl (we run it using `pw-jack qjakckctl` on pipewire). You probably wanna use a patchbay since capturing a monitor will replicate other people's voices too, while on a patchbay you can connect individual programs to what Discord captures.

## The Plan
It's a pretty simple one and emanates from the paragraph above. Since Chromium cannot capture a monitor we'll capture a microphone, merge it with the Screenshare stream using JavaScript and then connect the audio of the apps we want using a patchbay like helvum or qjackctl.

## The Problems
As mentioned in the prologue there are still some issues to be resolved.
1. Screenshare is only 720p 30fps. This cannot be fixed by forcing frameRate, width and height Video constraints.
2. Audio of both the microphone and desktop streams is mono. Forcing the channelCount: 2 constraint solely doesn't fix it. While Chromium recognizes it as stereo, Discord downmixes it.
3. This doesn't work on the Discord electron client. Electron uses getUserMedia to share your screen instead of getDisplayMedia, yet the official Discord client doesn't do any of these. As noted [here](https://blog.discord.com/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc-ce01c3187429), Discord uses a native module called MediaEngine that takes care of all input and output for their desktop and mobile clients which makes it difficult to do anything without a foss drop-in replacement.
4. This method is also problematic on Firefox, see bellow.

## The script
Here's the Javascript code used to achieve screensharing with audio, you probably wanna use something like Violentmonkey to load it for you.
```Javascript
// ==UserScript==
// @name         Screenshare with Audio
// @namespace    https://github.com/edisionnano
// @version      0.1
// @description  Screenshare with Audio on Discord
// @author       Guest271314 and Samantas5855
// @match        https://discord.com/*
// @icon         https://www.google.com/s2/favicons?domain=discord.com
// @grant        none
// @license      MIT
// ==/UserScript==
navigator.mediaDevices.chromiumGetDisplayMedia =
  navigator.mediaDevices.getDisplayMedia;
async function getDisplayMedia(
  { video: video, audio: audio } = { video: true, audio: true }
) {
  let captureSystemAudioStream = await navigator.mediaDevices.getUserMedia({
    audio: {
        // We add our audio constraints here, to get a list of supported constraints use navigator.mediaDevices.getSupportedConstraints();
        // We must capture a microphone, we use default since its the only deviceId that is the same for every Chromium user
        deviceId: { exact: "default" },
        // We want auto gain control, noise cancellation and noise suppression disabled so that our stream won't sound bad
        autoGainControl: false,
        echoCancellation: false,
        noiseSuppression: false,
        // By default Chromium sets channel count for audio devices to 1, we want it to be stereo in case we find a way for Discord to accept stereo screenshare too
        channelCount: 2,
        // You can set more audio constraints here, bellow are some examples
        latency: 0,
        sampleRate: 48000,
        sampleSize: 16,
        volume: 1.0
    }
  });
  let [track] = captureSystemAudioStream.getAudioTracks();
  const gdm = await navigator.mediaDevices.chromiumGetDisplayMedia({
    video: true,
    audio: true,
  });
  gdm.addTrack(track);
  return gdm;
}
navigator.mediaDevices.getDisplayMedia = getDisplayMedia;
var gdm = await navigator.mediaDevices.getDisplayMedia({
  audio: true,
  video: true,
});
//Stop the fake screenshare after getting permission
gdm.getTracks().forEach(track => track.stop());
```
For this to work, you need to make sure Discord doesn't capture the microphone called Default in your language, change that on Discord's Voice & Video settings.<br>
The script is hosted on [OpenUserJS](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio/source) and disables Chromium's awful processing only for the stream. If you want to disable them for your microphone too you can do so from Discord's Voice & Video settings or you can use [this](https://openuserjs.org/scripts/samantas5855/WebRTC_effects_remover) Violentmonkey script that disables them globally.<br>
When you launch Discord from https://discord.com/app or https://canary.discord.com/app or https://ptb.discord.com/app you will be presented with a dialog that asks for your permission to screenshare, it is important that you allow it so that to merge the sound of the default microphone with the desktop video, then you can start streaming on any call you may please.

## What about Firefox?
Firefox is my browser of choice so getting this to work on it was a priority for me. I've actually gotten pretty close to getting it to work without issues; while screenshare with desktop audio works it's pretty hard to use your microphone on the call. Firefox is a bit more secure that Chromium and actually follows the spec a bit more than it. So unlike Chromium, Firefox doesn't have a default device so we have to capture another input device, thankfully Firefox can list monitors so we capture these. Firefox also doesn't allow us to capture an input device more than once at the same time but we resolved this by automatically stopping the fake screenshare after getting permission. Firefox won't allow us to read the input device labels unless we get getUserMedia permissions. It doesn't allow us to call getDisplayMedia from the console unless we trick it by clicking the screenshare button on Discord first and then calling it from the console. And to top it all off Firefox doesn't seem to support the deviceId constraint even tho it's listed on `navigator.mediaDevices.getSupportedConstraints()`.<br>
Using the script bellow I was able to screenshare with audio on Discord by choosing my monitor however both my Discord microphone stream and the Desktop stream had the same audio (the monitor) and I couldn't find a way to fix that. Pavucontrol also won't work, probably because Firefox has multiple processes.
```Javascript
navigator.mediaDevices.chromiumGetDisplayMedia = navigator.mediaDevices.getDisplayMedia;
async function getDisplayMedia({video: video, audio: audio} = {video: true, audio: true}) {
  let captureSystemAudioStream = await navigator.mediaDevices.getUserMedia({audio: true});
  let [track] = captureSystemAudioStream.getAudioTracks();
  const gdm = await navigator.mediaDevices.chromiumGetDisplayMedia({video: true, audio: {autoGainControl: false, echoCancellation: false, noiseSuppression: false}});
  gdm.addTrack(track);
  return gdm;
}
navigator.mediaDevices.getDisplayMedia = getDisplayMedia;
var gdm = await navigator.mediaDevices.getDisplayMedia({audio: true, video: true});
gdm.getTracks().forEach(track => track.stop());
```
