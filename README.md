# **Offline Speech Recognition**

![](resources/thumbnail.png)

This is Unreal Engine plugin for accurate speech recognition, and it doesn't require internet connection.

# Table of contents
- [**Offline Speech Recognition**](#offline-speech-recognition)
- [Table of contents](#table-of-contents)
- [High level overview](#high-level-overview)
- [Project settings](#project-settings)
- [Test your microphone](#test-your-microphone)
- [Where to download languages and how to test them](#where-to-download-languages-and-how-to-test-them)
- [Using built-in language server](#using-built-in-language-server)
  - [Automatic speech recognition based on silence detection](#automatic-speech-recognition-based-on-silence-detection)
  - [Push to talk (Speak first, then recognize)](#push-to-talk-speak-first-then-recognize)
- [Running language server as external process](#running-language-server-as-external-process)
- [Running server process and game process at the same time](#running-server-process-and-game-process-at-the-same-time)
- [Passing SoundWave as input, instead of microphone](#passing-soundwave-as-input-instead-of-microphone)
  - [How Send Data to Language Server node works](#how-send-data-to-language-server-node-works)
- [Platforms supported](#platforms-supported)
- [Links](#links)


# High level overview
Since this is the speech to text plugin (STT), first thing you need is to be able to record your voice (any recording device). Then recorded voice is passed to speech recognizer, speech recognizer is giving your speech back in textual form. Speech recognizer is working with 1 language at a time. Each language is a downloadable folder with files.

In order to package shipe you game or app to end user, you will need to package each language model with your game, as well as language server itself (this is optional, since your game itself can be a server).


# Project settings
To make microphone work, you need to add following lines to `DefaultEngine.ini` of the project.
```
[Voice]
bEnabled=true
```

To not loose pauses in between words, you probably want to check silence detection threshold `voice.SilenceDetectionThreshold`, value `0.01` is good.
This also goes to `DefaultEngine.ini`.

```
[SystemSettings]
voice.SilenceDetectionThreshold=0.01
```
Starting from Engine version 4.25 also put
```
voice.MicNoiseGateThreshold=0.01
```

Another voice related variables worth playing with
```bash
voice.MicNoiseGateThreshold
voice.MicInputGain
voice.MicStereoBias
voice.MicNoiseAttackTime
voice.MicNoiseReleaseTime
voice.MicStereoBias
voice.SilenceDetectionAttackTime
voice.SilenceDetectionReleaseTime
```

To find available settings type `voice.` in editor console, and autocompletion widget will pop up.

![](resources/voicesettings.png)

Console variables can be modified in runtime like this

![](resources/silencenode.png)

Above values may differ depending on actual microphone characteristics.

# Test your microphone
To debug your microphone, input you can convert output sound buffer to
unreal sound wave and play it.

![](resources/buffertosound.png)

Another thing to keep in mind, if component connected to server, by default, it will try to send voice data during microphone capture. If you don't want this behavior, you can disable it like this

![](resources/send_data_when_recording.png)

Use this for push to talk style recognition (*when you record whole phrase first, and then send it to server*)

![](resources/push_to_talk_send_once.png)

# Where to download languages and how to test them
All available languages are available [here](https://alphacephei.com/vosk/models)

To test how specific language behaves, you can use [external language server app](https://github.com/IlgarLunin/vosk-language-server)

# Using built-in language server
*This method is preferable for simple scenarios, when you don't need to separate your game and language server, here you don't have all this hustle managing external process and communicating with server via web sockets.*

For both automatic and push to talk style recognition, you start from adding **SpeechRecognizer** component to your actor
   
![add_speech_recognizer](resources/add_speech_recognizer.png)

And then loading language into it. (This is non blocking function, and you know exactly when model is fully loaded into memory by connecting to **Finished** output pin)

![](resources/initialize_recognizer.png)

## Automatic speech recognition based on silence detection
![](resources/recognizer_automatic.png)

## Push to talk (Speak first, then recognize)
![](resources/recognizer_push_to_talk.png)

Feed voice data node can handle any amount of pre recorded speech, see [this section](#how-send-data-to-language-server-node-works)

# Running language server as external process
*In more complex cases this method is preferable over using built-in. You can have a single language server running in cloud or local server, and it can process multiple clients at the same time, since it's multithreaded.*

1. Download latest version [here](https://github.com/IlgarLunin/vosk-language-server/releases)
2. Run **vls.exe**, which is a user interface for **asr_server.exe**
   > **NOTE**: *asr_server.exe* is real server, you can run it without gui
   ![](resources/cli.png)
3. Go to main menu -> File -> Download models
   
   ![](resources/vlsdownloadmodels.png)

4. You will be redirected to a web page where you will find all available models (**languages**)
   
   ![](resources/modelspage.png)

5. In order to start using language, first download one of them
6. Enter path to downloaded model to server UI and press **start** button
   
   ![](resources/vlsrun.png)

   > **!NOTE!**: Depending on model size, you need to wait until model loaded in to memory, before start feeding server with voice data. e.g. If model size is ~2GB, it acn take ~10-30 seconds. But this is one time event, you can load your language to memory once with OS startup.
   ![](resources/ram.png)

7. Open unreal
8. Create actor blueprint
9. Add Vosk component in components panel

    ![](resources/addcomponent.png)

10. On begin play
    1. Bind to "Partial Result Received" event
    ![](resources/partialresult.png)

    1. **[Optional]** Bind to "Final Result Received" event
    ![](resources/finalresult.png)

    1. **[!MANDATORY!]** Connect to language server process and begin voice capture
    ![](resources/initialize.png)
    NOTE: `Addr` and `Port` coresponds to language server UI (*0.0.0.0 is the same as 127.0.0.1, it's just localhost*)
    ![](resources/initnode.png)


11. Start talking
12. Check *Partial Result Received* event gets executed

# Running server process and game process at the same time
Plugin offers following nodes

![](resources/server_process.png)

**Build Server Parameters** - helper method to simplify passing arguments to create process node

**Create Process** - Runs external program, this one is generic, you can use it to run whatever external program

 *NOTE*: *When you ship your game, you need to include language server as well, put language server files in your game bin folder (`GAME/Binaries/Win64/**`), and use "GetProcessExecutablePath" node to build path to `asr_server.exe`*

![](resources/process_path.png)

**Kill Process** - This is an equivalent of `Alt+F4`, it will shut down external process based on Process ID, the process id is process handle. Save output of `Create Process` node to a variable and use it later to terminate process.

Default use case:

* Create an `Actor` responsible for voice recognition
* Start language server on `Begin Play` event
* Add `Vosk` actor component and initialize it in begin play
* Begin capturing voice data
* Bind to message receive events
* Uninitialize vosk component and terminate server process on end play

> **NOTE**: *`Uninitialize` will stop voice capture if it is active*

![](resources/default_use_case.png)

# Passing SoundWave as input, instead of microphone

To do so, plugin offers a node that will convert sound into array of bytes, it is called `"Decompress Sound"`. You can than use output of decompress sound node in `"Send Voice Data to Language Server"` node, and expect partial and final result events being invoked later, when server finishes recognition.


> **NOTE**: *Do not call `BeginCapture` and `FinishCapture` in this case, since we don't want to use audio from the microphone*


![](resources/pass_sound_wave.png)

## How Send Data to Language Server node works
It takes sound bytes as first argument, and packet size as second argument. It will split all bytes into packets of given size, and send them one after another to language server, emulating microphone capture behavior. If packet size is greater than size of voice data, data will not be sent. 4096 packet size works relatively fast and suitable for short phrases. Note that if packet size is small, it will take more time to deliver entire voice to the server, and server will perform more iterations accordingly. You should play around with packet size in your specific case.


# Platforms supported

Tested on **Windows**



# Links

Find out more in documentation

* [Vosk](https://alphacephei.com/vosk/)
