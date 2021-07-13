# Summary

Capture Handle is a mechanism that allows a display-capturing web-application to ergonomically and confidently identify the web-application it is display-capturing (provided that the captured application has opted-in). Such identification allows these two applications to collaborate in interesting ways.

For example, if a VC application is capturing a presentation, then the VC application can expose user-controls for previous/next-slide directly in the VC application. This lets the user navigate presentations without having to jump between the VC and presentation tabs.

# Problem Description

## Generic Problem Description

Consider a web-application, running in one tab, which we’ll name “main_app.” Assume main_app calls [getDisplayMedia ](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) and the user chooses to share another tab, where an application is running which we’ll call “captured_app.”

Note that:

1. main_app does not know what it is capturing.
2. captured_app does not know that it is being captured; let alone by whom.

Both these traits are desirable for the general case, but there exist legitimate use cases where the browser would want to allow applications to opt-in to bridging that gap and enable a connection.

We wish to enable the legitimate use cases while keeping the general case as it was before.

## Use-case #1: Cross-App Communications

Consider two applications that wish to cooperate, for example a VC app and a presentation app. Assume the user is in a VC session. The user starts sharing a presentation. Both applications are interested in letting the VC app discover that it is capturing a slides session, which application, and even which session, so that the VC application will be able to expose controls to the user for flipping through slides. When the user clicks those controls, the VC app will be able to send messages to the presentation app (either through a service worker or through a shared back-end infrastructure). These messages will instruct the presentation app to flip through slides, enter/leave presentation-mode, etc.

## Use-case #2: Avoiding “Hall of Mirrors”

The “Hall of Mirrors” effect occurs when users choose to share the tab in which the VC call takes place. When detecting self-capture, a VC application can avoid displaying the captured stream back to the user, thereby avoiding the dreaded effect. Other mitigation strategies are also possible based on Capture Handle.

## Use-case #3: Detecting Unintended or Unapproved Captures

Users sometimes choose to share the wrong tab. Sometimes they switch to sharing the wrong tab by clicking the share-this-tab-insead button by mistake. A benevolent application could try to protect the user by presenting an in-app dialog for re-confirmation, if they believe that the user may have made a mistake.

## Use-case #4: Analytics

Capturing applications often wish to gather statistics over what applications their users tend to capture. For example, VC applications would like to know how often their users share presentation applications from specific providers, Wikipedia, CNN, etc. Gathering such information can be used to improve service for the users by introducing new collaborations, such as the one described above.


# Our Solution

## Summary

* Captured applications opt-in to exposing information by setting CaptureHandleConfig.
* Capturing applications read this information as CaptureHandle, which is available through [two access points](https://docs.google.com/document/d/1oSDmBPYVlxFJxb7ZB_rV6yaAaYIBFDphbkx5bXLnzFg/edit#bookmark=kix.y4ww0mbjjv2x).

## MediaDevices.setCaptureHandleConfig

```
dictionary CaptureHandleConfig {
  boolean exposeOrigin = false;
  DOMString handle = “”;
  sequence<DOMString> permittedOrigins = [];
};

partial interface MediaDevices {
  void setCaptureHandleConfig(
      optional CaptureHandleConfig config = {});
};
```

We add the method setCaptureHandleConfig in [MediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices). It accepts a configuration consisting of three independent members.

* **exposeOrigin:** If an application sets this value to true, the origin of that application may be exposed to capturing applications as CaptureHandle.origin.
* **handle:** If an application sets this value, that value is exposed to capturing applications as CaptureHandle.handle. Otherwise, the capturing application will see the empty string in that field. Values to this field are limited to 1024 unicode code points. If the application attempts to set a longer value, a TypeError exception is raised.
* **permittedOrigins:** A capturing application is only allowed to observe CaptureHandle if its origin is included in permittedOrigins. If permittedOrigins includes “*”, all capturers are permitted to observe CaptureHandle. In either of these cases, *origin* and *handle* are still exposed independently. That means that if *exposeOrigin* is false, capturers only see the handle. Defaulting to the empty set means that by default, nobody will see the capture handle, and the call will have no effect. This ensures that the caller has made an explicit decision on which origins to expose the capture handle to.

### Calls from an Embedded Frame

When *setCaptureHandleConfig()* is called from a document which is not the top-level document, an error is thrown.

### Side Effects

Whenever a captured application calls *setCaptureHandleConfig()*:

1. An event is fired on the capturer side. It has the new CaptureHandle as a property.
2. Any subsequent calls to [MediaStreamTrack.getSettings()](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack/getSettings) on the relevant tracks will produce a new [MediaTrackSettings](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings) object with a new CaptureHandle object.

### Empty CaptureHandleConfig

To clarify, the empty CaptureHandleConfig is the one which has all values set to their defaults.

## The CaptureHandle Type

```
dictionary CaptureHandle {
  DOMString origin;
  DOMString handle;
};
```

CaptureHandle is the object through which a capturing application may read information about the application it is capturing. It contains the following independent fields:

* **origin:** If the captured application opted-in to exposing its origin (by setting *CaptureHandleConfig.exposeOrigin* to true), then *CaptureHandle.origin* is set to the origin of the captured application. Otherwise, the CaptureHandle.origin is not set.
* **handle:** Reflects the value which the captured app set in *CaptureHandleConfig.handle*.

Capturing applications have two points of access to CaptureHandle objects:

1. Through *MediaTrackSettings.captureHandle*.
2. Through *CaptureHandleUpdateEvent*.

To clarify, the empty CaptureHandle is defined as that where both origin and handle are set to the empty string. If the captured application sets the empty CaptureHandleConfig, then the capturing application will read the empty CaptureHandle.

## MediaTrackSettings.captureHandle

```
partial dictionary MediaTrackSettings {
  CaptureHandle captureHandle;
};
```

Assume capturer is the application in the current tab, and captured is an application running in another tab. Assume capturer is display-capturing the tab in which captured lives, and that track, of type [MediaStreamTrack](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack), is either a video or an audio track associated with this capture. Calling track.[getSettings()](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack/getSettings) returns a [MediaTrackSettings](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings) object with a new field, captureHandle, of type *CaptureHandle*.

## CaptureHandleUpdateEvent

```
[Exposed=Window]
interface CaptureHandleUpdateEvent : Event {
  constructor(CaptureHandleUpdateEventInit);
  [SameObject] readonly CaptureHandle captureHandle;
};
```

A CaptureHandleUpdateEvent is fired in the capturing application’s JS context whenever the CaptureHandleConfig is updated in the captured application:

* When the captured application calls *MediaDevices.setCaptureHandleConfig* and sets a new configuration.
* When the captured tab’s top-level application is navigated away from a site that has set a non-empty capture handle.
* If the user manually changes the captured-tab, assuming the new site has a different CaptureHande (as observable by the capturing app).

The event has a single property - captureHandle - containing the capture-handle as it was at the time the event was fired. (This in contrast to the current handle, which is accessible via *getSettings*. When multiple events are fired at rapid succession, each will contain its respective value, and only the last one can be guaranteed to be equal to that exposed by getSettings.)

If an application tries registering an event handler on a track that’s originating from the browsing context in which the application is running, an error should be raised and the handler value should remain unchanged.

CaptureHandleUpdateEventInit is basically a replica of CaptureHandle, as is the pattern for event constructors in WebIDL.

## MediaStreamTrack.oncapturehandlechange

```
partial interface MediaStreamTrack {
  attribute EventHandler oncapturehandleupdate;
};
```

Allows capturing applications to register an event-handler. The handler is associated with a specific track, and therefore only receives the particular *event* associated with that track.

## CaptureHandle Asynchronicity

* MediaStreamTrack.getSettings().captureHandle returns the latest observable value.
* CaptureHandleUpdateEvent objects contain (as a property) the CaptureHandle at the time the event was fired.

## Navigation of the Captured Application’s Tab

When the captured tab’s top-level document is navigated cross-document, before navigation occurs, if a capture handle is set, the browser implicitly resets it. This fires an event.

Corollaries:

1. Assume capture begins of an application with handle=”a”. Assume navigation then unloads the application and replaces it with another application which sets handle=”b”. Two events will be fired. The first, upon navigating away, will be associated with an empty CaptureHandle. The second, once the new site loads and sets handle=”b”, will be associated with that new value.
2. Navigation away from a site that sets a CaptureHandle is detectable by the capturer.

Navigation away from a site that did not set a CaptureHandle is not detectable by the capturer. (More accurately - not more easily detectable than before.)

# Privacy + Security Considerations

## Opt-In
By default, nothing is revealed.

## User-driven
The process is still user-driven, as [getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) itself is user-driven - the user chooses what to share, as well as whether to share.

## Theoretically No-op
When users consent to a capture, they implicitly allow the capturing-app to receive any information that the captured-app wishes broadcast. Such information could be broadcast by embedding “magic pixels” in the captured-app, embedding QR codes in the video stream, etc.

The change described in this document is almost a no-op from the perspective of security and privacy. The word “almost” is necessary because previously the capturing-app would have to either take that information with a grain of salt, or validate it externally, e.g. by establishing contact with the collaborating app over some external medium. Now, at least one piece of information is mediated by the browser and therefore known to be non-spoofable - the origin.

## Captured-App Still Cannot Discover it is Being Captured
This change does not allow a captured application to discover when it is being captured, unless the capturing application sends it an out-of-band message to that effect. This would have been possible regardless of the change described in this document, and is therefore not a concern.

**Note:** The general concern with capturing applications discovering they’re being captured is that they could use that fact in order to censor their own content, limiting the user’s control and aggravating them. This concern does not apply to capturers who choose to inform the captured application of the capture. Such applications could either stop informing the captured application, after all. And the user is at the mercy of the capturing application to begin with, which could have chosen to never even start the capture.

## Controlled Exposure
One concern is that a capturer could misbehave when capturing specific origins. For example, when a VC application detects the user is capturing a competitor’s productivity suite, it could display ads for its own productivity suite. This concern is mitigated through the CaptureHandleConfig’s permittedOrigins field, which allows applications to control which origins may observe the CaptureHandle they set.

## App-controlled Opaqueness
Applications can use any CaptureHandle.handle they wish, and may also independently choose whether to expose their origin. The handle can either expose information according to some advertised format, or it can be completely opaque to anyone but a few privileged, collaborating apps.

For example, HypotheticalSite could make it widely known that their format is “HypotheticalSite:<rand_guid>”. This could then be used in tandem with some other API exposed by HypotheticalSite, such as an API for remotely controlling a playback (subject to access-control set by HypotheticalSite on their own API).

Continuing with the example above, it would also be possible for HypotheticalSite to set their format to <rand_guid>, with rand_guid as a 32-character hexadecimal string, and set exposeOrigin=false. It would then be difficult for arbitrary capturers to find out if they’re capturing a HypotheticalSite tab, since many different applications could follow that handle pattern. However, HypotheticalSite could give select collaborating applications access to a HypotheticalSite-operated API that checks whether a given GUID is a valid HypotheticalSite ID.

## Non-spoofable Origin
If the captured-application opts into exposing its origin, the capturing application gains access to a field that is non-spoofable. Claims made in the capture-handle can be trusted by the capturer if they are known to originate from a given origin. If an application chooses to share a handle but not the origin, on the contrary, the capturing application can either treat that information as suspect, or verify it in some external way before using it.

## Improvements over Steganography
As previously mentioned, applications could have previously used QR codes or [steganographic means](https://en.wikipedia.org/wiki/Steganography) to advertise some capture-handle. However, that would have been susceptible to interference from embedded frames, either intentionally or not. The capture handle mechanism, in contrast, is only accessible to the top-level document, and is safe from interference.

## Capturer Can Detect Navigation
* Assume EXP is a site exposing something - possibly the origin, possibly a handle, possibly both. When we want to denote two sites exposing different configs, we’ll name them EXP1 and EXP2. 
* Assume sites NOEXP, NOEXP1, NOEXP2, etc. never call setCaptureHandleConfig. (Recall that this is treated as implicitly calling setCaptureHandleConfig with the empty config.)

We distinguish these types navigation events:
1. NOEXP to EXP: Non-exposing site to exposing site.
2. EXP to NOEXP: Exposing site to non-exposing site.
3. EXP1 to EXP2: One exposing site to another, different exposing site.
4. EXP1 to EXP1*: One exposing site to another, but which sets the same configuration.
5. NOEXP1 to NOEXP2: One non-exposing site to another non-exposing site.

#1 partially reveals navigation. Depending on additional parameters, the capturer might have either full or partial certainty over whether navigation occurred, or whether EXP set a capture handle relatively late.

#2 and #3 make navigation unconcealable by definition.

#4 is similar to #2 in its first stage. Recall that when navigating away from EXP1 to EXP1*, the browser does not know if/when EXP1* will call setCaptureHandleConfig, and must treat navigation away from EXP1 as an implicit call to setCaptureHandleConfig with the empty config.

#5 We don’t fire an event in this case (rationale). The result of this decision is that a capturer can detect navigation away from an exposing site, but not navigation away from a non-exposing site.

## Excessive Events
It is possible for a captured application to “bombard” its capturer with events by repeatedly calling setCaptureHandleConfig. Note that this is true regardless of whether the event fires only on a new config, or whenever setCaptureHandleConfig is called, since the application can alternately set two different handles. This concern does not seem significant, as:
1. The captured application would be expending the same general amount of resources as it would be costing the capturing application. (Note that the case of multiple capturers is very rare in practice, and even then is limited to only a handful of capturers.)
2. The captured application would normally be carrying out the attack without knowing whether it is being captured, let alone knowing by whom. (The case where the capturer communicated its identity back to the captured application presumes a level of collaboration that makes such an attack unlikely, and at any rate - within the power of the capturing application to avoid.)

## Incognito Mode
Calls to setCaptureHandleConfig from an incognito tab must not be blocked, so as to avoid exposing incognito-status to the application. However, we avoid propagating the actual CaptureHandle between the capturing app and the captured app.
