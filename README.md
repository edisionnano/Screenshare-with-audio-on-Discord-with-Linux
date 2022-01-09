# Screenshare with audio on Discord with Linux
This repo allows you to Screenshare on Discord with Audio on Linux, on the web app without mixing Screenshare audio and microphone. It does so by redefining Chromium's getDisplayMedia. Technically, this fixes Chromium's ability to Screenshare/Screen Capture with Audio in general so it should work on every other video conferencing service that allows for Screensharing from a browser, if you know of any service other than Discord that does so please open an issue.

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
4. This method is also problematic on Firefox, see below.

## The script
Below, the Javascript code used to achieve Screensharing with Audio:
```Javascript
// ==UserScript==
// @name         Screenshare with Audio
// @namespace    https://github.com/edisionnano
// @version      0.3
// @description  Screenshare with Audio on Discord
// @author       Guest271314 and Samantas5855
// @match        https://*.discord.com/*
// @icon         https://www.google.com/s2/favicons?domain=discord.com
// @grant        none
// @license      MIT
// ==/UserScript==
navigator.mediaDevices.chromiumGetDisplayMedia =
  navigator.mediaDevices.getDisplayMedia;

const getDisplayMedia = async () => {
  let captureSystemAudioStream = await navigator.mediaDevices.getUserMedia({
    audio: {
      // We add our audio constraints here, to get a list of supported constraints use navigator.mediaDevices.getSupportedConstraints();
      // We must capture a microphone, we use default since its the only deviceId that is the same for every Chromium user
      deviceId: { exact: "default" },
      // We want auto gain control, noise cancellation and noise suppression disabled so that our stream won't sound bad
      autoGainControl: false,
      echoCancellation: false,
      noiseSuppression: false
      // By default Chromium sets channel count for audio devices to 1, we want it to be stereo in case we find a way for Discord to accept stereo screenshare too
      //channelCount: 2,
      // You can set more audio constraints here, below are some examples
      //latency: 0,
      //sampleRate: 48000,
      //sampleSize: 16,
      //volume: 1.0
    }
  });
  let [track] = captureSystemAudioStream.getAudioTracks();
  const gdm = await navigator.mediaDevices.chromiumGetDisplayMedia({
    video: true,
    audio: true
  });
  gdm.addTrack(track);
  return gdm;
};
navigator.mediaDevices.getDisplayMedia = getDisplayMedia;
```
For this to work, you need to make sure Discord doesn't capture the microphone called Default in your language, change that on Discord's Voice & Video settings.<br>
The script is hosted on [GreasyFork](https://greasyfork.org/en/scripts/436013-screenshare-with-audio) and [OpenUserJS](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio/) and disables Chromium's awful processing only for the desktop stream. If you want to disable them for your microphone too you can do so from Discord's Voice & Video settings or you can use [this](https://openuserjs.org/scripts/samantas5855/WebRTC_effects_remover) userscript that disables them globally.<br>

## How to use it
1. Make sure you use a Chromium-based browser like Chromium or Brave (Opera, Edge, Chrome and Vivaldi also fall under that category but aren't Open Source thus not recommended).
2. Install a UserScript manager like [Violentmonkey](https://chrome.google.com/webstore/detail/violentmonkey/jinjaccalgkegednnccohejagnlnfdag).
3. Click the install button to get the script on [GreasyFork](https://greasyfork.org/en/scripts/436013-screenshare-with-audio) or [OpenUserJS](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio/).
4. Make sure you have at least one physical or virtual input device, like a microphone, as this is needed for Chromium to list the "Default" device. If unsure, you can check with pavucontrol (works on both PulseAudio and PipeWire). Normally, if you have a microphone jack port on your PC and no microphone is plugged it should still count as an input device.
5. Open the Discord web app from one of the following links. [Normal](https://discord.com/app), [Canary](https://canary.discord.com/app) and [Public Test Build (PTB)](https://ptb.discord.com/app); all work.
6. If Chromium asks you, allow the microphone to be captured.
7. Go to Discord's audio settings and make sure the selected microphone isn't the one called "Default" (name will be different depending on the language your browser is on).
Join a call and start Screensharing.
8. There are two Chromium processes capturing audio, the first one is your microphone channel and the second one that appears is for the Screenshare stream. Your Screenshare's audio is now your microphone, but you obviously don't want that, here's how to change it:
    * If you want to share your desktop's full audio (that includes the voices of other people talking on the call) you can use pavucontrol which works on both PulseAudio and PipeWire; simply go to the recording tab and change Chromium to capture your monitor (you'll see two Chromium processes you may have to test to find out which one is which).
    * If you want to share audio of specific app(s) or full desktop audio excluding Chromium check the section below.<br>

Tips:
* Use the terminal command `pactl info` to check whether you use PulseAudio or PipeWire. If you see `Server Name: pulseaudio` you are on PulseAudio, if you see something along the lines of `Server Name: PulseAudio (on PipeWire X.XX.XX)` you are on PipeWire.
* If you are on Wayland and can't Screenshare on Chromium make sure you are on PipeWire and get the dependencies listed [here](https://wiki.archlinux.org/title/PipeWire#WebRTC_screen_sharing), package names may differ on other distros.
* To check whether you are on Wayland or X11 use the command `echo $XDG_SESSION_TYPE`.
* You can enable desktop notifications on the web app too, check Discord settings.
* If you want to install themes and plugins like on BetterDiscord, check out the [GooseMod](https://chrome.google.com/webstore/detail/goosemod-for-web/clgkdcccmbjmjdbdgcigpocfkkjeaeld) Chromium addon.

## Messing with audio
After you've configured the script you'll probably want to stream some audio but without capturing Chromium's output.  Since there's no dedicated app for this yet, we can do it easily with the terminal.<br>
</br>
**Case A: Capturing all desktop audio minus Chromium**
1. First we make sure that our terminal is english by running<br>
`export LC_ALL=C`
2. Then we'll create a parameter for our default output<br>
`DEFAULT_OUTPUT=$(pactl info|sed -n -e 's/^.*Default Sink: //p')`
3. We'll create two sinks, one for Chromium's audio and the other for the rest of your desktop audio.<br>
`pactl load-module module-null-sink sink_name=chromium`<br>
`pactl load-module module-null-sink sink_name=desktop_audio`
4. Now open your Chromium-based browser and make sure it's playing audio, you can load a YouTube video.<br>
Then run `pactl list sink-inputs` and copy the sink id corresponding to your Chromium based browser.
5. We want to move the browser to the chromium sink so we run<br>
`pactl move-sink-input $INDEX chromium`
where $INDEX is the index id from the previous command.<br>
You only need to do this once since the next time you create the chromium sink it will remember automatically to throw your browser in.<br>
    * On PulseAudio you can disable this behaviour by running
`pactl unload-module module-stream-restore`<br>
before the move-sink-input command.
6. Next we want to make sure that all other audio goes to the desktop_audio sink we created so we run<br>
`pactl set-default-sink desktop_audio`
    * On PulseAudio in order for your default sink to come back when you restart it then before running this you should run<br>
`pactl unload-module module-default-device-restore`<br>
on PipeWire this isn't needed
7. And finally we redirect the browser's and the rest of the audio to our physical output<br>
`pactl load-module module-loopback source=desktop_audio.monitor sink=chromium`<br>
`pactl load-module module-loopback source=chromium.monitor sink=$DEFAULT_OUTPUT`
8. Now we have to create a virtual microphone that mirrors our desktop's audio minus our browser's
     * On PulseAudio:<br>
`pactl load-module module-remap-source master=desktop_audio.monitor source_name=virtmic source_properties=device.description=virtmic`
    * On PipeWire:<br>
`nohup pw-loopback --capture-props='node.target=desktop_audio' --playback-props='media.class=Audio/Source node.name=virtmic node.description="virtmic"' >/dev/null &`
9. After you've finished screensharing you can reset everything by running
    * On PulseAudio:<br>
`pulseaudio -k`
    * On PipeWire:<br>
`systemctl --user restart pipewire`

After you've done this once you can then create a file called desktop.sh including<br>

```Shell
export LC_ALL=C
DEFAULT_OUTPUT=$(pactl info|sed -n -e 's/^.*Default Sink: //p')
if ! pactl info|grep -w "PipeWire">/dev/null; then
    pactl unload-module module-default-device-restore
fi
pactl load-module module-null-sink sink_name=chromium
pactl load-module module-null-sink sink_name=desktop_audio
pactl set-default-sink desktop_audio
pactl load-module module-loopback source=desktop_audio.monitor sink=$DEFAULT_OUTPUT
pactl load-module module-loopback source=chromium.monitor sink=$DEFAULT_OUTPUT
if pactl info|grep -w "PipeWire">/dev/null; then
    nohup pw-loopback --capture-props='node.target=desktop_audio' --playback-props='media.class=Audio/Source node.name=virtmic node.description="virtmic"' >/dev/null &
else
    pactl load-module module-remap-source master=desktop_audio.monitor source_name=virtmic source_properties=device.description=virtmic
fi
```

and run it using `sh desktop.sh` every time you want to screenshare with your whole desktop's audio.<br>
This should work for both PulseAudio and PipeWire.

Case A Tips:
* If your Chromium-Based browser is Chromium then throwing it on the chromium sink may also throw some electron apps although you probably don't care about sharing electron apps.
* While PipeWire is better than PulseAudio, compatibility with it isn't exactly perfect yet. You may need to throw Chromium to the chromium sink yourself using PavuControl sometimes.

**Case B: Sharing audio from one (or more) specific application(s)**<br>
Most stuff from above still apply on this case but this one is a bit simpler as we move apps seperately instead of everything minus one app.
1. We start by running:<br>
`export LC_ALL=C`<br>
to set our terminal to English.
2. And continue by saving the name of our physical audio output to a parameter with the following command:<br>
`DEFAULT_OUTPUT=$(pactl info|sed -n -e 's/^.*Default Sink: //p')`<br>
we are going to use this later in order to be able to listed to the app(s) whose audio we're sharing.
3. We're also going to create a SINK_NAME parameter so that you can copy the commands without having to replace the sink name each time. We only have to make one sink this time, give it an easy name corresponding to the app(s) you are going to throw inside so that you can load it easier the next time too.<br>
For example, since I'm going to name it `minecraft`, I should run:<br>
`SINK_NAME=minecraft`<br>
You should replace the value of `minecraft` with the sink name of your choice.
4. We now have to make the sink that our app(s) will be thrown into, simply run:<br>
`pactl load-module module-null-sink sink_name=$SINK_NAME`<br>
The first time we do this the app (in this case Minecraft) has to be running and output audio, then we'll have to find its ID and throw it on the sink on step 5. To make our life easier PulseAudio remembers this so the second time we'll load the sink named `minecraft` it will place the app there automatically even if it's not running or playing audio. This is very handy cause you can have multiple configurations that you can load on the fly. If, for some reason, you want to disable this feature on PulseAudio (not for PipeWire users) run:<br>
`pactl unload-module module-stream-restore`
5. You now have to make sure that the apps you want to share are running and are playing audio, then run:<br>
`pactl list sink-inputs`<br>
and find the Sink Input ID(s) of the application(s) that you want to share.
6. Move the applications to the sink using the `pactl move-sink-input` command. For example, Minecraft has Sink Input ID #1 and I want to move it to the minecraft sink. The appropriate command would be:<br>
`pactl move-sink-input 1 $SINK_NAME`<br>
and if I wanted to share another app with Sink Input ID #2 simultaneously with Minecraft I'd run:<br>
`pactl move-sink-input 1 $SINK_NAME`<br>
You can do this for as many or as little applications you wish, just make sure to replace the Sink Input ID(s) on the example command.
7. Considering you moved the application(s) correctly you should stop hearing them, to fix this we loop the audio from the sink back to our physical audio output by running:<br>
`pactl load-module module-loopback source=$SINK_NAME.monitor sink=$DEFAULT_OUTPUT`<br>
Now you should be able to hear it again.
8. The final step now is to create a virtual microphone that carries the sound that we want to share
     * On PulseAudio:<br>
`pactl load-module module-remap-source master=$SINK_NAME.monitor source_name=virtmic source_properties=device.description=virtmic`
    * On PipeWire:<br>
`nohup pw-loopback --capture-props='node.target='$SINK_NAME --playback-props='media.class=Audio/Source node.name=virtmic node.description="virtmic"' >/dev/null &`
9. You can reset everything after you've finished by running
    * On PulseAudio:<br>
`pulseaudio -k`
    * On PipeWire:<br>
`systemctl --user restart pipewire`

After finishing the tutorial successfully at least once you can create a shell script to do everything with just one command. You can name it `application.sh` and add the following

```Shell
SINK_NAME=
export LC_ALL=C
DEFAULT_OUTPUT=$(pactl info|sed -n -e 's/^.*Default Sink: //p')
pactl load-module module-null-sink sink_name=$SINK_NAME
pactl load-module module-loopback source=$SINK_NAME.monitor sink=$DEFAULT_OUTPUT
if pactl info|grep -w "PipeWire">/dev/null; then
    nohup pw-loopback --capture-props='node.target='$SINK_NAME --playback-props='media.class=Audio/Source node.name=virtmic node.description="virtmic"' >/dev/null &
else
    pactl load-module module-remap-source master=$SINK_NAME.monitor source_name=virtmic source_properties=device.description=virtmic
fi
```

Then either give `SINK_NAME=` a value, have different shell scripts for every sink and run them using `sh application.sh` (replace application.sh if you named it otherwise) OR delete the first line and call the script with a parameter, for example for me that I made the minecraft sink I'd do<br>
`SINK_NAME=minecraft sh application.sh`

Case B Tips:
* If you throw an Electron app inside the sink and your browser is Chromium then it may throw that inside too resulting in other users in the call hearing themselves back from the stream. However this is a rare since you probably don't want to share Electron apps.
* If you are on PipeWire chances are you'll have to throw the app(s) you want to share the sound of in the sinks every time.

**Getting the script to use virtmic**<br>
If you've followed the tutorial of one of the two cases, chances are that you ended up with a virtual microphone called virtmic that carries the sound you desire to share. However, the script doesn't pick virtmic, it instead picks the input device created by Chromium called default in your language. The reason is simple, that device always exists as long as the system has one input device and its id is always default. If we wanted to explicitly pick a specific device, while possible with JavaScript some issues exist; first of all the virtmic device has to exist, secondly Discord should already have microphone permission and thirdly we have to make sure that in Discord's audio settings neither Default nor virtmic are used as our microphone source. If you have done all these, you can uninstall the script and use [this](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio_(virtmic)) version I've specifically created for this case.

## What about Firefox?
Firefox is my browser of choice so getting this to work on it was a priority for me. I've actually gotten pretty close to getting it to work without issues; while screenshare with desktop audio works it's pretty hard to use your microphone on the call. Firefox is a bit more secure that Chromium and actually follows the spec a bit more than it. So unlike Chromium, Firefox doesn't have a default device so we have to capture another input device, thankfully Firefox can list monitors so we capture these. Firefox also doesn't allow us to capture an input device more than once at the same time but we resolved this by automatically stopping the fake screenshare after getting permission. Firefox won't allow us to read the input device labels unless we get getUserMedia permissions. It doesn't allow us to call getDisplayMedia from the console unless we trick it by clicking the screenshare button on Discord first and then calling it from the console. And to top it all off Firefox doesn't seem to support the deviceId constraint even tho it's listed on `navigator.mediaDevices.getSupportedConstraints()`.<br>
Using the script below I was able to screenshare with audio on Discord by choosing my monitor however both my Discord microphone stream and the Desktop stream had the same audio (the monitor) and I couldn't find a way to fix that. Pavucontrol also won't work, probably because Firefox has multiple processes.
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

If you are interested in Firefox support, follow [this issue](https://bugzilla.mozilla.org/show_bug.cgi?id=1238038).

## Still have questions?
Contact me at Samantas5855#2607 on Discord for additional support.
