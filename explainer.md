# Audio Focus API Explainer

*Last modified: Feb 13th 2019*

## Objectives
People consume a lot of media (audio/video) and the Web is one of the primary means of consuming this type of content. However, media on the web does not integrate well with the platform. The Audio Focus API helps to close the gap with platforms that have audio focus such as Android and iOS. This API will help by improving the audio-mixing of websites with native apps, so they can play on top of each other, or play exclusively.

Additionally, on some platforms the user agent will automatically manage audio focus for the site based on whether media elements are playing or not. In some cases this may not match user expectations so this API will provide overrides for the authors.

### Goals

 * **A site should be able to manage its own audio focus.** If a site wishes to manage its own audio focus then the user agent should not automatically manage it. This would be used on a site where the default user agent audio focus logic is not appropriate (a media site where we switch tracks) or supported (e.g. WebAudio, WebRTC sites).
 * **A site should be able to define how audio streams will interact with the platform.** This is where it can be annoying where two tabs play audio at the same time. However, in some cases it may be appropriate to play the two audio streams on top of each other (e.g. a transient ping).
 * **A site should be able to determine its own audio focus state.** A site should be notified if its audio focus changes. This is so sites that are manually managing focus can be aware of their current state.
 * **To provide an experience on par with native apps.** Native apps have audio focus APIs on some platforms so we should provide a similar level of experience for websites.

### Non-goals

* We should aim to improve the audio interaction between the site and the platform. Audio interaction within the site should be up to the site.

## API Design

Firstly, audio focus means an audio-producing object is allowed to play sound. By convention, there are several `audio focus types` for different purposes:

 * Playback (`playback`) audio, which is used for video or music playback, podcasts, etc. They should not mix with other playback audio. (Maybe) they should pause all other audio indefinitely.
 * Transient (`transient`) audio, such as a notification ping. They usually should play on top of playback audio (and maybe also "duck" persistent audio).
 * Transient solo (`transient-solo`) audio, such as driving directions. They should pause/mute all other audio and play exclusively. When a transient-solo audio ended, it should resume the paused/muted audio.
 * Ambient (`ambient`) audio, which is mixable with other types of audio. This is useful in some special cases such as when the user wants to mix audios from multiple pages.

The AudioFocusSession is the main interface for this API. It can have the following states:

 * active: the AudioFocusSession is allowed to play sound.
 * suspended: the AudioFocusSession is not allowed to play sound, but can resume when it regains focus.
 * inactive: the AudioFocusSession is not allowed to play sound, and will not regain focus unless it requests audio focus again.

The page can have a default AudioFocusSession which is used by the user agent to automatically request and abandon audio focus when media elements start/finish playing on the page. This session is created automatically by the user agent when the page is loaded.

```javascript
enum AudioFocusState {
  "inactive",
  "active",
  "suspended"
};

enum AudioFocusType {
  "playback",
  "transient",
  "transient-solo",
  "ambient"
};

[Exposed=Window]
interface AudioFocus {
  // The default audio focus session that the user agent will use
  // when media elements start/stop playing. This will be created
  // by the user agent when the page is loaded. If set to null then
  // this feature will be disabled.
  attribute AudioFocusSession? defaultSession;
};

[Exposed=Window, Constructor(AudioFocusType type)]
interface AudioFocusSession : EventTarget {
  // Request audio focus from the platform and the boolean will be
  // true if the request was successful.
  Promise<bool> request();

  // Abandons audio focus. Throws an error if the session does
  // not have audio focus.
  Promise<void> abandon();

  readonly attribute AudioFocusState state;
  attribute EventHandler onchange;
};
```

There should only be one audio focus session active on a page at one time. If there are multiple sessions on a page then when one requests audio focus it will make all the other sessions inactive.

### Sample Code

#### A site manages its own audio focus

In this situation a site (e.g. a game) wants to manage its own audio focus. In this case it can use the following code to create an AudioFocusSession and request/abandon audio focus:

```javascript
// Prevent the user agent from managing focus
audioFocus.defaultSession = null;

const session = AudioFocusSession(‘transient’);

session.request().then(
 (success) => {  /* handle request result */ },
 (e) => {  /* internal error */ });

session.abandon().then(
 () => {  /* audio focus was abandoned */ },
 (e) => {  /* could not abandon audio focus */ });

session.addEventListener(‘onchange’, (e) => // state change);
```

#### A site would like to customise it’s audio focus

In this situation a site would like to use the default audio focus logic provided by the user agent. However, they would like to customise the type of audio focus to request. Therefore, they create a custom audio focus session and assign it to the defaultSession variable. 

```javascript
audioFocus.defaultSession = AudioFocusSession(‘transient’);

// 1. Site starts playing some transient media
// 2. User now clicks a video

audioFocus.defaultSession = AudioFocusSession(‘ambient’);
audioFocus.defaultSession.addEventListener(
    ‘onchange’, (e) => // state change);

// Site plays the video and the user agent will automatically
// manage audio focus.
A site would like to observe audio focus state
If a site would like to simply observe the audio focus state. They can create an event listener on the default session of the page.

audioFocus.defaultSession.addEventListener(
    ‘onchange’, (e) => // state change);
```

#### A site is playing a combination of media types

If a site would like to play a combination of media types (e.g. a video and a notification) then they should change the default session of the page and manually duck their own video.

```javascript

// A user is playing a video on a site and receives a notification 
// ping.

audioFocus.defaultSession = AudioFocusSession(‘transient’);

// All other sessions are ducked. We should also duck the video
// element on the page by manually adjusting the volume.

video.volume = 0.8;

// When the ping is done playing we can change the focus back to
// playback and the playing video will join that session.

audioFocus.defaultSession = AudioFocusSession(‘playback’);
```

This is an alternative implementation without using the default session:

```javascript
audioFocus.defaultSession = null;

// A user starts playing a video on a site.
const session = AudioFocusSession(‘playback’);
session.request();

// The user receives a notification ping. In this case calling
// request on |transientSession| will make |session| inactive.
const transientSession = AudioFocusSession(‘transient’);

transientSession.addEventListener(
    ‘onchange’, (e) => {
       // If the session becomes active then all other audio on
       // the system is ducked so we should also manually duck
       // our video element.
       video.volume = transientSession.state == ‘active’ ? 0.8 : 1.0;
     });

transientSession.request();

// When the ping is done playing we can change the focus back by
// calling request on the first session since this will make
// |transientSession| inactive.
session.request();
```
