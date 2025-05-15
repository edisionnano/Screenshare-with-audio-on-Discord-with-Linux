# Screenshare with audio on Discord with Linux
This repo allows you to Screenshare on Discord with Audio on Linux, on the web app without mixing Screenshare audio and microphone. It does so by redefining your browser's getDisplayMedia function. Technically, this fixes the browser's ability to Screenshare/Screen Capture with Audio in general so it should work on every other video conferencing service that allows for Screensharing from a browser like Jitsi.

## For beginners
Applications which incorporate this tutorial and automate its steps exist to provide a user-friendly experience. These are the following:
* [discord-screenaudio](https://github.com/maltejur/discord-screenaudio) is a simple app that uses QTWebEngine, it's available on the AUR as well as FlatHub
* [Vesktop](https://github.com/Vencord/Vesktop) is an app by Vencord which supports Vencord plugins and themes, it uses Electron to wrap the browser version of Discord into an app. Vesktop is available as an AppImage but can also be found on the AUR
* [pipewire-screenaudio](https://github.com/IceDBorn/pipewire-screenaudio) is a two-piece Firefox addon and system app which simplifies the process for those using Gecko based browsers to screenshare
* [My simple GUI](https://gist.github.com/edisionnano/4cd912315ae4e309261147be23ed0dee) is a very straightforward script which uses Zenity to create an app selector dialog, it will also download the virtmic binary for you. Zenity is preinstalled on SteamOS but you may need to install it on other distributions.

## How to use it
1. Install a UserScript manager like [Violentmonkey](https://violentmonkey.github.io/get-it/).
2. Click the install button to get the script on [GreasyFork](https://greasyfork.org/en/scripts/436013-screenshare-with-audio) or [OpenUserJS](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio/).
3. Open the Discord web app from one of the following links and login. [Normal](https://discord.com/app), [Canary](https://canary.discord.com/app) and [Public Test Build (PTB)](https://ptb.discord.com/app); all work.
4. Go to Discord's audio settings and make sure the selected microphone isn't the one called "Default" (name will be different depending on the language your browser is on). If your browser asks you, allow the microphone to be captured.
5. Use the terminal command `pactl info` to check whether you use PulseAudio or PipeWire. If you see `Server Name: pulseaudio` you are on PulseAudio, if you see something along the lines of `Server Name: PulseAudio (on PipeWire X.XX.XX)` you are on PipeWire. If you use PipeWire your life becomes easier since you can use the tool that automates the process, if you are on regular PulseAudio you'll have to stop at this step and follow the manual steps mentioned [here](https://github.com/edisionnano/Screenshare-with-audio-on-Discord-with-Linux#messing-with-audio).
6. Type `bash <(curl -s https://gist.githubusercontent.com/edisionnano/4cd912315ae4e309261147be23ed0dee/raw/e03e739e1aee407f2163af096604752f8166ed5c/discord.sh)` on your terminal to run my GUI and select a sound source, you can also alias this or save the script locally to run it faster if you want.
7. Make sure both Violentmonkey and the userscript are enabled, then start screensharing either your whole screen or a specific app and sound should work.

Tips and notes:
* If you are on Wayland and can't Screenshare on Chromium make sure you are on PipeWire and get the dependencies listed [here](https://wiki.archlinux.org/title/PipeWire#WebRTC_screen_sharing), package names may differ on other distros.
* To check whether you are on Wayland or X11 use the command `echo $XDG_SESSION_TYPE`.
* You can enable desktop notifications on the web app too, check Discord settings.
* The tool's source code can be found [here](https://github.com/Curve/rohrkabel/tree/a092c91fe773ea968d98874a604acc31b82531bf/examples/link-app-to-mic), huge thanks to Curve for this.
* Rohrkabel is licensed under the MIT license which can be found [here](https://github.com/Curve/rohrkabel/blob/master/LICENSE).
* You can also choose a virtmic source without my GUI, just download the tool from [here](https://github.com/edisionnano/Screenshare-with-audio-on-Discord-with-Linux/blob/main/virtmic?raw=true), make it executable using `chmod +x virtmic` and execute it using `./virtmic`. The tool now asks you for the name of the app you want to share, to find the name of the app you'll have to use `pw-cli ls Node` while the app is running. Once you type it simply hit enter and minimize the terminal.
* Violentmonkey and other script managers rely on Manifest V2 support, you can re-enable this manually on Chromium-based browsers, check [here](https://github.com/uBlockOrigin/uBlock-issues/discussions/2977#discussioncomment-9521603).

## Still have questions?
Contact me at @samantas5855 (was Samantas5855#2607) on Discord for additional support.<br>
Continue reading if you want to know more about how this works.

## Prologue
Screensharing on Discord has been a huge pain for non Windows users since desktop audio would not be captured and until recently there was no option to even pick one screen if you had multiple. Screensharing with desktop audio was recently fixed on Mac OS on the official client only through a proprietary hack since getting desktop audio on Mac isn't easy and you need something like Soundflower to interact with the kernel and electron/chromium don't have such functionality.<br>
A hack many people do is to mix their microphone with their desktop audio to stream it that way, this is not proper because all users on the call are forced to listen to your desktop audio regardless of whether they are watching your stream or not (a huge issue when many people are streaming), they cannot adjust your stream and microphone volume individually, mic channels overlap on discord when many people are talking, bitrate is low, the sound is mono and effects are applied. Some of these issues may appear in our methods too but it's possible to fix them using JavaScript which is why this repo exists in the first place.

## Enter Chromium
As you probably know Discord's client isn't a native app, it's a web browser since it uses electron that is a modified version of Chromium. You can use Chromium or any Chromium-Based browser like Brave and Vivaldi to log in on Discord and you can even Screenshare but desktop audio will only work on Windows and Linux with Chromium OS Audio Server (so only Chrome/Chromium OS). Linux and Mac users can only share the audio of a tab.<br>
As discussed in the prologue it's hard to get desktop audio on Mac OS because a signed kernel extension is needed so it must be hard on Linux too right? The answer is no, in fact, any app that can capture an input can also capture desktop audio, that's because on PulseAudio (includes PipeWire) there exist some virtual input devices called Monitors. Monitors replicate the audio of an output (like your speakers or headphones) and make it an input. Chromium used to block the capture of Monitor devices but the block has been lifted, while not enabled by default the flag #pulseaudio-loopback-for-screen-share toggles this behaviour. Using a Monitor as Screenshare can be helpful on some services but not Discord since the other users will listen to themselves on the Screenshare stream's audio. If you are familiar with Linux's audio server you can control what Chromium captures through pulseaudio assistant programms like pavucontrol or a patchbay like helvum and qjackctl (we run it using `pw-jack qjakckctl` on pipewire). A patchbay will let you connect the outputs of processes like games to Chromium's Screenshare input using nodes.

## The Plan
It's a pretty simple one and emanates from the paragraph above. Since Chromium cannot capture a monitor we'll capture a microphone, merge it with the Screenshare stream using JavaScript and then connect the audio of the apps we want using a patchbay like helvum or qjackctl.

## The Problems
As mentioned in the prologue there are still some issues to be resolved.
1. Screenshare is only 720p 30fps. This cannot be fixed by forcing frameRate, width and height Video constraints.
2. This doesn't work on the Discord electron client. Electron uses getUserMedia to share your screen instead of getDisplayMedia, yet the official Discord client doesn't do any of these. As noted [here](https://discord.com/blog/how-discord-handles-two-and-half-million-concurrent-voice-users-using-webrtc), Discord uses a native module called MediaEngine that takes care of all input and output for their desktop and mobile clients which makes it difficult to do anything without a foss drop-in replacement.

## The script
Below, the Javascript code used to achieve Screensharing with Audio:
```Javascript
// ==UserScript==
// @name         Screenshare with Audio
// @namespace    https://github.com/edisionnano
// @version      0.6
// @updateURL    https://openuserjs.org/meta/samantas5855/Screenshare_with_Audio.meta.js
// @description  Screenshare with Audio on Discord
// @author       Guest271314 and Samantas5855
// @match        https://*.discord.com/*
// @icon         https://www.google.com/s2/favicons?domain=discord.com
// @grant        none
// @license      MIT
// ==/UserScript==

/* jshint esversion: 11 */

(async () => {
    'use strict';

    async function getVirtMicTrack() {
        await navigator.mediaDevices.getUserMedia({
            audio: true
        });
        await new Promise(r => setTimeout(r, 500));
        const devices = await navigator.mediaDevices.enumerateDevices();
        const virtmic = devices.find(d => d.kind === "audioinput" && d.label.toLowerCase().includes("virtmic"));
        if (!virtmic) {
            return null;
        }
        const stream = await navigator.mediaDevices.getUserMedia({
            audio: {
                deviceId: {
                    exact: virtmic.deviceId
                },
                channelCount: 2,
                autoGainControl: false,
                echoCancellation: false,
                noiseSuppression: false
                // You can set more audio constraints here, bellow are some examples
                // just don't forget to add a comma at every option other than the last
                //latency: 0,
                //sampleRate: 48000,
                //sampleSize: 16,
                //volume: 1.0
            }
        });
        return stream.getAudioTracks()[0];
    }

    const hookDisplayMedia = async () => {
        if (!navigator.mediaDevices?.getDisplayMedia) {
            return;
        }

        const original = navigator.mediaDevices.getDisplayMedia;
        navigator.mediaDevices.getDisplayMedia = async function(constraints) {
            const displayStream = await original.call(navigator.mediaDevices, constraints);
            const virtmicTrack = await getVirtMicTrack();
            if (!virtmicTrack) return displayStream;
            displayStream.getAudioTracks().forEach(track => displayStream.removeTrack(track));
            displayStream.addTrack(virtmicTrack);
            return displayStream;
        };
    };

    let retries = 0;
    const interval = setInterval(() => {
        if (retries++ > 10) {
            clearInterval(interval);
            return;
        }
        if (navigator.mediaDevices?.getDisplayMedia) {
            clearInterval(interval);
            hookDisplayMedia();
        }
    }, 500);

    //Stereo support thanks to @dakrk
    const originalSetRemoteDescription = RTCPeerConnection.prototype.setRemoteDescription;
    RTCPeerConnection.prototype.setRemoteDescription = function(...args) {
        if (args[0]?.sdp) {
            args[0].sdp = args[0].sdp.replaceAll('useinbandfec=1', 'useinbandfec=1;stereo=1');
        }
        return originalSetRemoteDescription.apply(this, args);
    };
})();
```
For this to work, you need to make sure Discord doesn't capture the microphone called Default in your language, change that on Discord's Voice & Video settings.<br>
The script is hosted on [GreasyFork](https://greasyfork.org/en/scripts/436013-screenshare-with-audio) and [OpenUserJS](https://openuserjs.org/scripts/samantas5855/Screenshare_with_Audio/) and disables your browser's awful audio processing only for the desktop stream. If you want to disable them for your microphone too you can do so from Discord's Voice & Video settings or you can use [this](https://openuserjs.org/scripts/samantas5855/WebRTC_effects_remover) userscript that disables them globally.<br>

## Messing with audio
After you've configured the script you'll probably want to stream some audio but without capturing your browser's output.  Since there's no dedicated app for this yet, we can do it easily with the terminal.<br>
</br>
**Case A: Capturing all desktop audio minus the browser**
1. First we make sure that our terminal is english by running<br>
`export LC_ALL=C`
2. Then we'll create a parameter for our default output<br>
`DEFAULT_OUTPUT=$(pactl info|sed -n -e 's/^.*Default Sink: //p')`
3. We'll create two sinks, one for your browser's audio (in this case Chromium) and the other for the rest of your desktop audio.<br>
`pactl load-module module-null-sink sink_name=chromium`<br>
`pactl load-module module-null-sink sink_name=desktop_audio`
4. Now start your browser and make sure it's playing audio, you can load a YouTube video.<br>
Then run `pactl list sink-inputs` and copy the sink id corresponding to your browser.
5. We want to move the browser to the sink we created for the browser so we run<br>
`pactl move-sink-input $INDEX chromium`
where $INDEX is the index id from the previous command.<br>
You only need to do this once since the next time you create the chromium sink (or whatever else you named it) it will remember automatically to throw your browser in.<br>
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
* If your browser is Chromium then throwing it on the chromium sink may also throw some electron apps although you probably don't care about sharing electron apps.
* While PipeWire is better than PulseAudio, compatibility with it isn't exactly perfect yet. You may need to throw your browser to the chromium sink yourself using PavuControl sometimes.

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
`pactl move-sink-input 2 $SINK_NAME`<br>
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
The script searches for virtmic by default and only if that's not found does it fallback to Default. Just make sure you have the latest version of the script and that neither Default nor virtmic are selected as input devices on Discord's "Voice & Video" settings.

## What about Firefox?
From Firefox version 101 onward the script works out of the box as long as a microphone named `virtmic` exists since Firefox has no device with id `default`. However when streaming from Firefox there do exist severe audio quality/pitching issues on 101, these issues were fixed on later versions.
