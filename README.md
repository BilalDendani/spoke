# spoke
The spoke project is Spoke's server-side library that interfaces with custom speech technologies and provides a utility-belt of audio tools such as recording, transcoding, and playback. [spoke][1] can be used as a standalone processing library or in conjunction with [spoke-client][2] to build websites.

## Installation
Prerequisites: You must have Node.js and NPM already installed. As well, for the audio conversions to work properly, you must have SoX installed on the server where spoke will be running.
Open your project's package.json file and add a dependency on spoke, linking to this github repo:
```js
  "dependencies": {
    ...
    "spoke": "git://github.com/psaylor/spoke",
    ...
  }
```
Then in your terminal you will want to run an update to complete the install
```sh
$ npm install
# or
$ npm update
```

# Spoke
Spoke is a framework for building interactive speech-enabled web applications. It leverages modern web technologies such as the Web Audio APIs, WebSockets, and Node.js to provide a standardized and streamlined way to build website demos featuring one's own spoken language technologies on the backend. 

Though Spoke was designed with the needs of the MIT Spoken Language Systems group in mind, it could easily be adopted by other researchers and developers hoping to incorporate their own speech technologies into functional websites.


Spoke consists of two components (each their own project with github repo) that should be used in conjunction to build web applications:

1. [spoke][1]: a Node.js server-side library with a set of modules that interface with a handful of custom speech technologies (such as speech recognition, forced alignment, and mispronunciation detection). 
2. [spoke-client][2]: a JavaScript client-side library and framework with a set of tools for building front-end speech recognition websites (such as recording audio to a server, visualizing audio information, speech recognition through browser APIs).

![Spoke module overview][Spoke_module_overview]

For more on [spoke][1], keep reading. [spoke-client][2] is explained in more detail on its own page.

## Overview
spoke, the server-side library of Spoke, provides two types of modules-**audio utilities** and **speech technologies**-geared towards building a speech processing backend for a server, but the library can also be used for offline scripting and batch processing. As a library, not a framework, it is up to the developer to listen for client events and then decide which pieces of the library to use. 

The library’s **utilities** enable 
* recording raw or wav audio
* transcoding audio files or live audio streams
* streaming audio from a saved file, and more

The library also provides standardized access to and usage of some **speech technologies** built in the lab, such as 
* domain-specific speech recognizers
* forced alignment of an audio sample to expected text
* mispronunciation detection on a set of utterances from the same speaker

It is important to note that any technology that has a command line interface can also be integrated into spoke to make it accessible for website backends and offline JavaScript processing. For technologies with complex outputs (such as forced alignment) the technology module is paired with an output parser that transforms the output into a usable JavaScript object.


### Design Pattern
Each module employs prototype-based object-oriented programming to export one object providing a clean interface to a tool or technology. Under the hood, most of these modules operate by running a command in a new process, though the utility modules do this indirectly through the [SoxCommand][sox-audio] interface. In Node’s single-threaded model, asynchronously creating these new processes is essential, allowing the developer to spin up a long running process and then wait for a callback; Node provides a [child_process module][3] to accomplish exactly this.  

Callbacks are the standard Node.js way, but they are not always the best way for handling asynchronous control flow when multiple asynchronous steps need to be chained together. In this case, the preferred option is that the command execution method returns a [Promise][4] instead of calling a callback, since Promises can easily be chained to create a sequential pipeline of asynchronous tasks . Each module in the library supports both styles, callback-based and Promise-based, leaving it to the developer’s discretion for her particular case. For these method pairs, the methods that return [Promises][4] will have the name of its callback method counterpart but with the suffix "Async" (e.g. `recognize` and `recognizeAsync`). Both methods are still asynchronous, but since the introduction of Promises the "Async" suffix for Promisified methods has become convention.



## Audio Utilities
spoke provides a utility-belt of audio tools that complement its speech technologies with support for recording audio streams to a file, creating audio streams from a file, and building arbitrarily complex [SoX][sox] commands that will be run in a new process. This last piece is enabled by [sox-audio], a Node.js package that, independent of Spoke, is useful for the community of Node.js developers, so it was released as a [standalone package][https://www.npmjs.com/package/sox-audio] on [NPM][npm]. The Recorder and Player modules then use [sox-audio] to do most of the heavy lifting.
### Utils
[utils.js] provides a few general-purpose helper functions that were useful for building Spoke websites, but it is by no means exhaustive. So far it simplifies parsing an XML file and normalizing strings to remove certain punctuation and make it lowercase. This module should be extended as more developers use spoke and find themselves using a general-purpose function in a few places to reduce code redundancy.

### Recorder
The [Recorder module][recorder.js] records audio from a raw input stream to a file, either in the original raw format or in an optionally downsampled wav format. This is primarily used to enable live audio recording on the web for collection and/or subsequent processing with one of the speech technologies.

At initialization, the Recorder can be configured with information about the raw input streams it will be handling, such as their bit depth and sample rate, and with the desired sample rate for converted wav files. The crucial method is convertAndSave, which accepts a raw audio file or raw audio stream and a wav file or stream for outputting the transcoded results; transcoding is carried out with a SoxCommand built from the method’s parameters for input and output streams or files and the Recorder configuration for sample rates, etc.  For offline processing, this can be used to transcode already saved raw files to wav files. More interestingly, as part of a web backend, this can be used to perform streaming transcoding of a raw stream to a wav stream, which could start being processed before all the audio has been received if a speech technology accepted streaming audio. As it is, all of them currently accept only saved wav files, but this should change over the next year.

#### Recorder Defaults and Configuration
```js
Recorder.DEFAULTS = {
    /* Default input configuration for raw input */
    inputEncoding: 'signed',
    inputBits: 16,
    inputChannels: 1,
    inputSampleRate: 44100,

    /* Default output configuration */
    outputFileType: 'wav',
    outputSampleRate: 16000,
};
```
Any of these properties can be configured when you initialize a new Recorder object and then they apply to all of the recorder's methods. The properties specifying information about the input or output audio are passed directly into a [SoxCommand][sox-audio], so you can read the [sox-audio] docs or [sox] docs to learn more about these properties and appropriate values for them.

These defaults fit most situations, but say you are recording a wav stream with a sample rate of 48 kHz and you want to downsample it to 8 kHz, then you can configure your recorder as follows:
```js
var Spoke = require('spoke');
var recOptions = {
  inputSampleRate: 48000,
  outputSampleRate: 8000,
}; 
var recorder = new Spoke.Recorder(recOptions);
```

#### Recorder Methods
* __`convertAndSave(rawInputAudio, outputAudio, cb)`__: Converts inputAudio, a raw audio file or raw audio stream, to outputFileType format with sample rate outputSampleRate (from options, defaults to a 16kHz wav format). You can provide a callback which will get passed an error if there was one, or the result if successful: cb(err, result). Returns this Recorder instance for chaining.
* __`convertAndSaveAsync(rawInputAudio, outputAudio)`__: Converts inputAudio, a raw audio file or raw audio stream, to outputFileType format with sample rate outputSampleRate (from options, defaults to a 16kHz wav format). Returns a Promise which will be fulfilled with the outputAudio stream or file upon completion, or rejected with the SoxCommand error upon failure.
* __`saveRaw(rawAudioStream, filename, cb)`__: Saves the provided raw audio stream to filename. You can provide a callback which will get passed an error if there was one, or the result if successful: cb(err, result). Returns this Recorder instance for chaining.
* __`saveRawAsync(rawAudioStream, filename)`__: Saves the provided raw audio stream to filename.eturns a Promise which will be fulfilled with the filename of the saved file upon completion, or rejected with the SoxCommand error upon failure.

#### Recording with Callbacks
```js
var Spoke = require('spoke');
var recorder = new Spoke.Recorder();
var rawFilename = 'input.raw';
var wavFilename = 'input.wav';

var callback1 = function () {
  console.log('Raw audio saved asynchronously');
};
recorder.saveRaw(stream, rawFilename, callback1);

var callback2 = function (err, savedFilename) {
  if (err) {
    console.log('Error converting the raw to a wav');
    return;
  }
  console.log('Recording completed:', savedFilename);
  // Optionally process using a speech technology
};
recorder.convertAndSave(stream, wavFilename, callback2);
```

#### Recording with Promises
```js
var Spoke = require('spoke');
var recorder = new Spoke.Recorder();
var rawFilename = 'input.raw';
var wavFilename = 'input.wav';

// Save the raw audio stream as is
recorder.saveRawAsync(rawAudioStream, rawFilename)
  .then(function () {
    console.log('Raw audio saved asynchronously with Promises');
  });
  
// Convert the incoming raw audio stream to a wav file and save it
recorder.convertAndSaveAsync(rawAudioStream, wavFilename)
  .catch(function reject (err) {
    console.log('Error converting the raw to a wav');
  })
  .then(function resolve (savedFilename) {
    console.log('Recording completed:', savedFilename);
    // Optionally process using a speech technology
  });
```
### Player

## Integrated Speech Technologies

### Recognizer
### Forced Alignment
### Mispronunciation Detection

[1]:https://github.com/psaylor/spoke "the spoke repo"
[2]:https://github.com/psaylor/spoke-client "the spoke-client repo"
[sox-audio]:https://github.com/psaylor/sox-audio "the sox-audio repo"
[sox]:http://sox.sourceforge.net/sox.html "SoX-the Swiss Army knife of audio manipulation"
[npm]:https://www.npmjs.com/ "npm registry"
[3]:https://nodejs.org/api/child_process.html "node.js child process docs"
[4]:https://www.promisejs.org/ "explanation of Promises"
[utils.js]:https://github.com/psaylor/spoke/blob/master/lib/utils.js
[recorder.js]:https://github.com/psaylor/spoke/blob/master/lib/recorder.js
[Spoke_module_overview]: https://github.com/psaylor/spoke-client/blob/gh-pages/images/Spoke_module_overview.png "Spoke module overview"
