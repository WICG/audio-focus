# AudioFocus API Explained

## Objectives

People consume a lot of media (audio/video) and the Web is one of the primary
means of consuming this type of content. One problem of Web media is that they
are usually badly mixed. The AudioFocus API helps improving the audio-mixing of
Web media, so they can play on top of each other, or play exclusively. This API
also improves the audio mixing between Web media and native applications.

## Motivation

The existing behavior when multiple tabs play sound at the same time and how
they interact with each other and native applications (on the platform) is
barely defined, which usually brings bad user experience. It can be annoying
when two tabs play video at the same time. However it is usually acceptable to
let a transient ping play above music playback.

For example, if the user plays a video on Facebook while listening to music in
another tab, the Facebook video should pause the music playback. However, if
Facebook wants to play a messenger notification, it will annoy the user if the
ping interrupt the music playback. Lowering the music volume and let the
Facebook ping play above the music will bring better experience.

## API design

The AudioFocus API design is still a sketch. Comments/suggestions are welcomed.

Firstly, **audio focus** means an audio-producing object is allowed to play
sound. By convention, there are several `audio focus types` for different
purposes:

* Playback (`playback`) audio *(The name was content previously)*, which is used
  for video or music playback, podcasts, etc. They should not mix with other
  `playback` audio. (Maybe) they should pause all other audio indefinitely.
* Transient (`transient`) audio, such as a notification ping. They usually
  should play on top of `playback` audio (and maybe also "duck" persistent
  audio).
* Transient solo (`transient-solo`) audio, such as driving directions. They
  should pause/mute all other audio and play exclusively. When a
  `transient-solo` audio ended, it should resume the paused/muted audios.
* Ambient (`ambient`) audio, which is mixable with other types of audios. This
  is useful in some special cases such as when the user wants to mix audios from
  multiple pages.

The audio focus types above may be different from audio focus types provided by
some platform. There might be compatibility issues for audio focus types in the
API and platform. For example, iOS does not explicitly distinguish `transient`
and non-`transient`, and Android does not explicitly have `ambient` type.
Therefore, the audio focus types in the API are not strict, and the user agent
should adapt them to platform conventions.

We list the possible solutions for Android & iOS here:

* On Android, we can use AudioManager.requestAudioFocus(), and specify
  `StreamType` x `DurationHint`, where:
  * StreamType can be MUSIC, NOTIFICATION, ALARM, etc.
  * DurationHint can be GAIN, TRANSIENT_MAY_DUCK, TRANSIENT
* On iOS, we can use AVAudioSession, and specify `AudioSessionCategory` x
  `AudioSessionCategoryOptions`, where:
  * AudioSessionCategory can be Ambient, SoloAmbient, Playback, Record, etc.
    AudioSessionCategoryOptions can have MixWithOthers, DuckOthers, etc.

The implementations on Android & iOS for the 4 audio focus types in the API
would be:

| Audio focus type | Android | iOS |
|------------------|---------|-----|
| `playback` | MUSIC x GAIN | Playback |
| `transient` | MUSIC/NOTIFICATION x TRANSIENT_MAY_DUCK | Playback/Ambient x DuckOthers |
| `transient-solo` | MUSIC/NOTIFICATION x TRANSIENT | Playback/SoloAmbient |
| `ambient` | Not requesting audio focus and respond or responding audio focus changes at all (may be bad)? | Playback/Ambient x MixWithOthers |

## The model for handling audio focus

`AudioFocusEntry` is the minimum unit for handling audio focus. An
`AudioFocusEntry` has a type indicating whether it is `playback`, `transient`,
`transient-solo` or `ambient`. Here we only consider media elements for
describing the model, other audio producing objects such as Flash, AudioContext
(WebAudio) and WebRTC will discussed later.

The page creates an `AudioFocusEntry` with an audio focus type, and it can
associate media elements to the `AudioFocusEntry`. When the page wants to play
audio, it needs to request audio focus through the AudioFocusEntry, either by
calling play() of the associated media element or explicitly requesting audio
focus by javascript (even without associating with any media elements). The user
agent needs to decide whether the request is successful and tell the
AudioFocusEntry. Then the element can play after the audio focus is granted.
Otherwise, the play request is rejected. AudioFocusEntrys may optionally tell
the user agent when it does not want to play anymore (abandon audio focus).

A sample snippet is as follows:

``` javascript
// Suppose |audio| is an <audio> element.
var focusEntry = new AudioFocusEntry("playback");
audio.focusEntry = focusEntry;
audio.play()
    .then(function () {
        console.log("play() success");
    }).catch(function () {
        console.log("failed to request audio focus");
    });
```

AudioFocusEntrys can have the following states:

* active: the `AudioFocusEntry` is allowed to play sound.
* suspended: the `AudioFocusEntry` is not allowed to play sound, but can resume
  when it regains focus.
* ducking: the `AudioFocusEntry` is allowed to play sound with reduced volume.
* inactive: the `AudioFocusEntry` is not allowed to play sound, and will not
  regain focus unless it requests audio focus again.

**Note** We may only want to expose only some of the states to the page. Maybe
only `active`, `suspended` and `inactive`? `Ducking` should be handled
internally in the user agent.

The user agent can perform the following operations to AudioFocusEntrys:

* Suspend: transform the AudioFocusEntry state from active to suspended.
* Resume: transform the AudioFocusEntry state from suspended or ducking to active.
* Inactivate: transform the AudioFocusEntry state from any other than inactive to inactive.
* Duck: transform the AudioFocusEntry state from active to ducking.

The user agent should keep track of all active, suspended and ducking
`AudioFocusEntrys`, which is put into its **managed AudioFocusEntry set**. There
is at most one active/suspended/ducking AudioFocusEntry of `playback` type. When
an AudioFocusEntry tries to request audio focus or abandons audio focus, the
user agent must decide whether to grant the play request and how to change the
state of the managed AudioFocusEntry set. The detailed implementation is up to
the user agent. Some example behaviors are:

* When there are no managed AudioFocusEntry set, and a `playback` type
  AudioFocusEntry request to play, the user agent should grant the play request
  and put the AudioFocusEntry into its managed AudioFocusEntry set.
* When there is currently an active `playback` type AudioFocusEntry A, and
  another `playback` type AudioFocusEntry B requests to play, the user agent
  should inactive A and grant the play request to B. Then the user agent should
  remove A from the managed AudioFocusEntries and put B into the managed
  AudioFocusEntries.
* When there is currently an active `playback` type AudioFocusEntry A, and
  another `transient` type AudioFocusEntry B requests to play, the user agent
  should duck A and grant play request to B. When B abandons audio focus, the
  user agent should resume A.

Besides, if the platform has audio focus handling mechanisms, the user agent
should behave as a proxy and forward the AudioFocusEntry requests to the
platform. The user agent should also listen to audio focus related signals
coming from the platform and update managed AudioFocusEntry states accordingly.
For example, if another native music app starts playback, and the platform will
tell the user agent it should suspend, then the user agent should suspend all
AudioFocusEntrys.

To summarize, the processing model can be described as follows:

* AudioFocusEntry is the minimum unit for handling audio focus.
* A media element should join an AudioFocusEntry when it wants to play audio.
* An AudioFocusEntry should tell the user agent when it wants to play audio or
  stops playing audio. This can be done implicitly when its media elements
  starts to play, or by the page explicitly activating the AudioFocusEntry.
* The user agent must keeps track of all non-inactive AudioFocusEntrys, and must
  update the state of managed AudioFocusEntrys when an AudioFocusEntry
  request/abandons audio focus.
* If the platform has audio focus handling mechanism, the user agent should act
  as a proxy and forwards audio focus signals between the pages and the
  platform.

## Handling WebAudio, Flash (and maybe WebRTC) Issue

The topic discussed in this section is still an open question.

WebAudio and Flash and WebRTC are not like media elements, they need to be
addressed differently.

### WebAudio

To let WebAudio participate in audio focus management, we can add a `focusEntry`
attribute to AudioContext to let WebAudio join AudioFocusEntry:

```
focusEntry = new AudioFocusEntry();
audioContext = new AudioContext();
audioContext.focusEntry = focusEntry();
```

When Webaudio wants to start playback, since we cannot really know when WebAudio
starts, the page is responsible to activate the AudioFocusEntry() explicitly:

```
focusEntry.activate()
  .then(function() {
    // Start WebAudio playback.
  }).catch(function(e) {
    console.log("cannot request audio focus for WebAudio");
  });
```

When responding to audio focus changes, the user agent should call
`AudioContext.suspend()` or `AudioContext.resume()` accordingly.

### Flash

Flash is similar with WebAudio, the playback cannot be controlled. Also, unlike
WebAudio, it is harder to let Flash join AudioFocusEntry since we might need to
modify <object> elements. Maybe we could have a default AudioFocusEntry per page
and let Flash join the default one.

### WebRTC

WebRTC is more complex, since it usually require `voice call` focus, and on
some platforms, the platform have to change the audio routing and trigger some
other complex behaviors. Besides, there are different use cases we need to
define the desired behavior, such as:

* When a media element is playing and WebRTC starts, we may prefer media element
  be suspended.
* When the user is using WebRTC to chat with a friend and then he/she opens a
  video recommended by his/her friend, the user might not want WebRTC to be
  paused/muted.

So maybe a *one-shot* audio focus type is needed for this case.

### Fallback behavior when the page does not use AudioFocus API

We need to define a behavior when the page does not use AudioFocus API. The
behavior should also be compatible with our model. There are several doable
ways:

* Having a global `playback` type AudioFocusEntry per page, and all media
  elements belong to the global AudioFocusEntry by default, or:
* Having a global `playback` type AudioFocusEntry and a global `transient` type
  AudioFocusEntry per page, all long media elements belong to the `playback`
  AudioFocusEntry, and all short media elements belong to the `transient`
  AudioFocusEntry. (This solution is more like the current Clank behavior).
