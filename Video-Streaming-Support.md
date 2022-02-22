While ring-mqtt is primarily designed to integrate Ring devices into home automation platforms via MQTT to allow driving automations from those devices, there was high demand to provide video streaming integration as well, especially for Home Assistant users, but also to provide features such as on-demand recording via automations.  With the release of version 4.8.0 video streaming and recording automation is now fully integrated into this project.

This document provides detailed information about the video streaming support, including how to configure it with Home Assistant or use it with other medial players, as well as some troubleshooting information and known limitations.  If you would like to use the videos streaming features, please read this section carefully.

**!!!! Important note regarding camera support !!!!**    
The ring-mqtt project does not turn Ring cameras into 24x7/continuous streaming CCTV cameras.  Ring cameras are designed to work with Ring cloud servers for on-demand streaming based on detected events (motion/ding) or short term (10 minute) interactive viewing/recording.  Even when using ring-mqtt, all streaming still goes through Ring cloud servers and is not local.  Attempting to leverage this project for continuous streaming is not a supported use case and attempts to do so will almost certainly end in disappointment, this includes use with NVR tools like Frigate or motionEye and there are significant functional side effects to doing so, most notably loss of motion/ding events while streaming (Ring cameras only send alerts when not streaming).  While there are some projects out there that attempt to use Ring cameras in this way by constantly restarting the stream after timeouts, this project will not be one of them unless Ring eventually allows a 24x7 viewing option.

### Quick overview
Ring video streaming support is implemented by running a local instance of rtsp-simple-server.  For each camera discovered two separate RTSP paths are registered with the server using the following format:

Live Stream:  <camera_id>_live  
Event Stream: <camera_id>_event

To start a stream all that is required is to use any media client that support the RTSP protocol to connect to the given URL.  Behind the scenes the rtsp-simple-server uses an on-demand script to communicate via MQTT to ring-mqtt to instruct it when to start and stop the video stream.

The "live" path always starts a live view stream for the camera, while the "event" path starts a stream of a previously recorded event.  By default the event stream plays the most recently recorded motion event, however you can use the event select feature to select any for the 5 most recent motion, ding (doorbells only), or on-demand recording events.  For more details see the event stream section below.

### Quick live stream configuration with Home Assistant
Due to the fact that MQTT is not a technology suitable for streaming, the MQTT camera support in Home Assistant only supports still images updated at most every 10 seconds and it is not currently possible to have Home Assistant automatically discover the video streaming cameras via the MQTT integration.  Because of this, the video streaming cameras will need to be configured manually in configuration.yaml.  Home Assistant provides a significant number of camera platforms that can work with RTSP streams, but this document will focus on the setup of the [Generic IP Camera](https://www.home-assistant.io/integrations/generic/) integration.

To setup the generic IP camera you will need to manually add entries to the Home Assistant configuration.yaml file.  How to do this is outside of the scope of this document so please read up on that if you are not familiar with editing the Home Assistant configuration files.

Setting up a camera requires a basic entry like this at a minimum:
```
camera:
  - platform: generic
    name: <Your Device Name Here>
    still_image_url: <image_url>
    stream_source: <stream_url>
```

Thie "name" option is the name that the camera will appear as in the Home Assistant UI.  You can use a URL to any image for the still_image_url but the suggested configuration is to enable the snapshot feature in this addon and use the camera proxy API so you get a nice, automatically updating still image.  The configuration example below does exactly that, providing an example of using a value template to pull the current MQTT snapshot image using the Home Assistant camera proxy API.  Using this setup, the picture glance card will display the most recent snapshot and simply clicking the image will open the video stream.

The stream_source is the URL required to play the video stream.  To make the camera setup as easy as possible, ring-mqtt attempts to guess the required entries and includes them as attributes in the camera info sensor.  Simply open the camera device in Home Assistant, select the Info sensor entity, and the open the attributes and there will be a stream source and still image URL entry that you can copy and paste to create your config.  Alternately, you can find the attributes using the Developer Tools portion of the UI and finding the info sensor entity for the camera.  While the addon makes efforts to guess the correct URL, because the addon runs as an entirely separate process from Home Assistant, it has limited information to build the exact URL so in many cases it will not be 100% correct.  When running as an addon the code does attempt to query the API to get more information, but it still may not get exact port and other data correct.

The following example uses a camera with the name "Front Porch" in the Ring app.  The MQTT discovered snapshot camera has a Home Assistant entity ID of **camera.front_porch_snapshot** and the camera device ID is **3452b19184fa** so the attributes in the info sensor are as follows:
```
Still Image URL: http://<MY_HA_HOSTNAME>:8123{{ states.camera.front_porch_snapshot.attributes.entity_picture }}  
Stream Source:   rtsp://03cabcc9-ring-mqtt:8554/3452b19184fa_live
```
To create the generic IP camera for the live video stream, in the configuration.yaml, simply add the following lines:
```
camera:
  - platform: generic
    name: Front Porch Video
    still_image_url: http://<MY_HA_HOSTNAME>:8123{{ states.camera.front_porch_snapshot.attributes.entity_picture }}
    stream_source: rtsp://03cabcc9-ring-mqtt:8554/3452b19184fa_live
```

Note that the still_image_url uses the guessed hostname and/or localhost.  This could work in some cases, but if SSL is enabled or the default port has been changed, the correct URL may not be reflected here, simply use the Home Assistant base URL with the value template on the end and make sure there is no slash after the hostname or port.  For example, if the Home Assistant instance is accessed directly via https://myha.mydomain.local/ then the URL would be https://myha.mydomain.local{{ states.camera.front_porch_snapshot.attributes.entity_picture }}.  If SSL is in use, but is using self-signed certificates, or if the use of localhost or the IP address instead of the full HA hostname is desired, then the `verify_ssl: false` config option will likely need to be added as well.  This is because the SSL certificate will typically be bound to the HA instance full hostname so attempts to connect via localhost/IP address will cause invalid certificate warnings.

Once the configuration is saved, simply reload the configuration (generic IP camera entities can be reloaded without a full HA restart) and the new camera entities should appear.  These cameras can now be added to the dashboard via a Picture Glance card or any other card that supports cameras.  With no special configuration this should now provide a card that provides still image snapshots based on the addon snapshot settings, and then, with a click, open a window that starts a live stream of that camera.

The picture glance card is quite flexible and it's possible to add the additional camera entities to this card as well, like the motion, light and siren switches (for devices with those features) and the stream switch.  With this setup it's possible to see at a glance if any motion/ding event is active, see the latest snapshot from that event, see the light/siren/stream state, and a simple click opens the live stream.

### Event Stream
** Please note that use of this feature requires a Ring Protect plan that supports video storage **

As mentioned above, this addon provides two separate paths for video streams, one that always provides a live stream, and a second that can stream a selected, previously recorded stream.  Camera setup for this feature is the same as above but uses the "<camera_id>_event" path vs the live path in the previous example.

On startup the default event is set to the most recent motion (Motion 1) but the play back event can be selected using the Event Selector entity in Home Assistant or the equivalent MQTT command topic.  Each camera allows selecting from any of the last five motion, ding, or on-demand events (ding events are available only for doorbells).  For example, selecting "Ding 1" will cause the event stream to play back the recording of the most recent doorbell ding event, while selecting "Motion 3" would play back the 3rd most recently recorded motion event.  On-demand recording events occur any time a video stream is started for on-demand viewing without a motion/ding event.

When a recorded stream is playing it is streamed only a single time for each RTSP client request and then the stream shuts down until the next request for a stream.  Stream playback can be manually stopped via the event stream switch, although note that, unlike live streams, playback of recorded events can only be started via on-demand viewing.

If a recorded stream is actively playing, changing the event selector immediately cancels playback of the existing event stream and the stream will not start again until a new client makes an RTSP request (for example, just closing and re-opening the playback window in Home Assistant).

### Authentication
By default, the addon does not expose the RTSP server to external devices so only Home Assistant can actually access the streams, thus using a non-authenticated stream isn't too bad since the stream stays completely internal to the Home Assistant server.  This assumes that the addon is running on the same server as Home Assistant.

However, it is completely possible to access the live stream via external media clients like VLC or running your own ffmpeg process, for example if you want to trigger a local recording or push an MJPEG stream to a TV.  However, if it is desired to expose the RTSP server, setting the **livestream_user/livestream_pass** configuration options (**LIVESTREAMUSER/LIVESTREAMPASSWORD** environment variables for standard Docker installs) is HIGHLY recommended prior to doing so.  Note that currently, even with a username and password, the stream is not encrypted, so using the stream over an untrusted network without a VPN is not recommended as is not a good idea.  Note that both username and password must be defined before this feature is enabled.

If a username and password is defined, then both publishers and streamers will require this password to access the RTSP server.  This is handled automatically for the stream publishing and the RTSP URL will include the username/password in the URL so that they can be included in the **configuration.yaml**.  A sample is as follows:
```
camera:
  - platform: generic
    name: Front Porch Live
    still_image_url: http://localhost:8123{{ states.camera.front_porch_snapshot.attributes.entity_picture }}
    stream_source: rtsp://streaming_user:let_me_stream!@3ba32cf2-ring-mqtt-dev:8554/3452b19184fa_live
```
### External RTSP Access
Streaming to external media clients requires the RTSP server to be exposed either via the addon configuration settings or via the Docker -p port forwarding option.  It's recommended to use TCP port 8554, but you can actually forward any external TCP port to the 8554 on the RTSP server in the container.  Note that streams will start automatically on-demand, and end ~5-10 seconds after the last client disconnects.  Multiple clients can connect to the same stream concurrently.  No MQTT access is needed for this to work, simply enter the RTSP URL into your media player.  If you defined a livestream username and password this will need to be included as well, most players will prompt for a username/password, but some require them to be included in the URL, for example:

`rtsp://streaming_user:let_me_stream!@3ba32cf2-ring-mqtt:8554/3452b19184fa_live`

### Manually Starting a Live Stream
When a media client connects to the live stream server the stream is started automatically, however, there may be cases where starting a stream without a media client is desired.  When used with a Ring Protect plan all live streams also create a recording so being able to manually start a stream allows an automation to effectively manually start and stop a recording.  This is possible utilizing the "live stream" switch, which is exposed as an entity to Home Assistant and can be accessed by MQTT as well.  It performs like any switch, accepting "ON" and "OFF" to start and stop a stream repsectively.

Note that turning the stream off ALWAYS stops the local live stream immediately, no matter how many clients are connected to the local RTSP server.

### Downloading recorded videos using the Event Stream
If this addon is used with an account that includes a Ring Protect Plan that supports saving videos to the Ring Cloud service, it is possible to use this addon to automate downloading of videos once they have been processed.  To assist with this, the "Select Event Stream" entity includes attributes for both the current eventId and the recordingUrl.

Note that recordingURLs are only valid for 15 minutes so the addon automatically requests a new URL around the 10 minute mark prior to the old URL expiring.  Also, any time a new event stream is selected the eventId and recordingUrl are immediately updated with the information for the selected event.  This means it's not a good idea to trigger downloads specifically on eventID changes.

As an alternative, the best method is to use an automation that triggers on the event type being downloaded, then use a wait for trigger to perform the download as soon as the eventId changes.  Below is a simple example automation that uses the Home Assistant downloader service to download a recording as soon as the eventId is updated, which indicates that the recording is ready.

```
alias: Download Ring Video (Front Porch)
trigger:
  - platform: state
    entity_id: binary_sensor.front_porch_motion
    to: 'on'
action:
  - wait_for_trigger:
      - platform: state
        entity_id: select.front_porch_event_select
        attribute: eventId
    timeout: '00:05'
  - service: downloader.download_file
    data_template:
      url: '{{ states.select.front_porch_event_select.attributes.recordingUrl }}'
      subdir: front_porch
      filename: '{{ now().strftime( ''%Y%m%dT%H%M%S_motion.mp4'' ) }}'
      overwrite: false
```

The automation in this example is initially triggered any time a motion event starts.  Once triggered, it waits for the eventId attribute to change, which indicates that the recording of the new event is ready. At that point it uses the Home Assistant downloader service with the recordingUrl attribute to download the file to a subdirectory with a date based filename.

Of course there are other possible automation options as well, and, even without a Ring Protect Plan, you can do things like start an FFmpeg stream on a motion event to record a video, however, the Ring Protect Plan still offers a significant value in that it pulls the seconds just before the event, while triggering a recording of the stream will always miss the first few seconds at least since it won't know to start recording until after the motion event is received.  Of course, if you are using other devices or events as the start trigger, this might be good enough.

### FAQ

**Q) Why do streams keep running for 5+ minutes after viewing them in Home Assistant**  
**A)** Home Assistant keeps streams running in the background for ~5 minutes even when they are no longer viewed.  It's always possible to stop streams manually using the stream switch.

**Q) Streams keep starting all the time even when I'm not viewing anything**  
**A)** In Home Assistant, do not use the "Preload Stream" option and make sure the Camera View setting in the Picture Glance card is set to "auto" instead of "live".  If you use the "live" setting, the card will attempt to start streams in the background for faster startup when you bring up the card.  This is fine for local cameras but, because Ring cameras do not send motion events during streams, having streams running all the time will cause motion events to be missed and, since all streaming goes through Ring servers on the Internet, you will use a lot of bandwidth as well.

There have also been reports of certain components connecting to live streams with when no password is set on the RTSP server, I've not personally seen this behavior but mulitple users have sent log examples showing "something" connecting to the RTSP server which triggers the stream to start.  The simplest way to solve this is to set a livestream username/password so that services which automatically discover RTSP endpoints cannot connect and start streams.

**Q) Why does the live stream stop after ~10 minutes?**  
**A)** Ring enforces a time limit on active live streams and terminates them, typically after approximately 10 minutes, although sometimes significantly less and sometimes a little more.  Currently, you'll need to refresh to manually restart the stream but it is NOT recommended to attempt to stream 24 hours.  Ring has hinted that continuous live streaming is something they are working on, but, for now, the code honors the exiting limits and does not just immediately reconnect to the stream.

**Q) Why is the stream delayed/lagged?**
**A)** The code path for streaming is quite optimized and typically adds less than one second of latency, however, Home Assistant uses the HLS protocol for streaming in the web browser which splits the existing stream into segments for delivery over HTTP/HTTPS.  While HLS streaming is extremely reliable and widely compatible with various web browsers and network setups, it typically adds 7-10 seconds of delay and, in some cases, significantly more, which is generally not great for live viewing. Fortuantely, starting with Home Assistant 2022.2, the streaming component defaults to using the low-latency variant of the HLS protocol (LL-HLS) which helps to reduce latency to 3-5 seconds in most cases, however, this can lead to an increase in artifacts and video stuttering, see the question below on video artifact/stuttering for more details on possible mitigations for this behavior.

For the lowest latency viewing possible, the best solution for Home Assistant is the use of a custom UI card like the excellent [WebRTC Camera](https://github.com/AlexxIT/WebRTC) which will allow you to use the native video playback capabilities of modern browsers to view the stream.  Unfortunately, WebRTC is a much more complex protocol vs HLS and may require special configuration to support play back while outside of your local network without using a VPN. however, WebRTC provides by far the highest quality, lowest latency streaming option available in the Home Assistant UI, generally providing mostly artifact free viewing with latency in the 1-2 second range.

Other options that offer low-latency viewing are the use an external media player capable of RTSP playback.  VLC works well, but note that it buffers 1 second of video by default, although on fast networks you can tweak this to as low as 0 milliseconds to reduce the delay even further.

**Q) Why do I have video artifacts and/or stuttering in the stream?**   
**A)** There are two likely sources of artifacts/stuttering and I'll outline both below:
- Ring streams include a significant number of minor encoding anomolies, especially in the first 5-10 seconds of the stream. At first I thought this was a bug in stream handling but further investigation showed that the same anomolies exist in the video recording file downloaded directly from Ring servers, so it clearly indicates the error is in the initial stream encoding from the camera.  These anomolies go largely unnoticed in media players that decode the raw RTP stream using protocols like RTSP and WebRTC, however, they cause issue with the Home Assistant stream component which uses specific markers to split the RTP stream into the required HLS segments and this is made even worse by the introduction of LL-HLS in Home Assistant 2022.2, which further chunks the streams into even smaller parts.  Options are to disable LL-HLS in the configuration, which minimizes the artifacts but increases latency, or, after much experimentation, the following settings have been found to minimize (not eliminate) the artifacts while keeping most of the benefit of LL-HLS only increasing the latency a second or so while significantly improving overall playback stability, especially in early stages of the stream:
```
stream:
  ll_hls: true
  segment_duration: 3.625
  part_duration: 1.45
```

- The second issue mostly impacts the live stream which uses RTP over UDP and these packets are then processed by ring-client-api inside of the NodeJS process before being sent via a pipe to ffmpeg.  While NodeJS is quite fast for an interpreted language like Javascript, it's still not exactly the most efficient for real-time stream processing so you need a reasonable amount of CPU available.  Having a good CPU and a solid networking setup that does not drop UDP packets is critical to reliable and artifact free function of the live stream.  If you have mulitiple cameras, or a system with limited CPU/RAM (RPi3 for example) then it will be difficult to support more than a handful of streams concurrently.  Testing shows that an RPi3 struggles to support more than 4 concurrent streams while an RPi4 can handle 6-7 concurrent live streams, and a decent Intel based machine can handle about 2-3 live streams per-core.  These numbers assume that the load on the CPU from other components is minimal.

**Q) Why do I see high memory usage?**  
**A)** Support for live streaming uses rtsp-simple-server, which is a binary process running in addition to the normal node process used by ring-mqtt.  When idle, this process uses minimal memory (typically <20MB), however, each active stream has at least one FFmpeg process to read the incoming stream and publish it to the server.  Total memory usage is typically about 25-30MB got each active stream on top of the base memory usage of the addon.  Also, when using Home Assistant, the Home Assistant core memory usage will also increase somewhat for each stream.

**Q) Why are there no motion events while live streaming?**  
**A)** This is a limitation of Ring cameras as they do not detect/send motion events while a stream/recording is active.  The code itself has no limitations in this regard.

**Q) Why do I have so many recordings on my Ring App?**  
**A)** If you have a Ring Protect subscription then all "live streams" are actually recording sessions as well, so every time you start a live view of your camera you will see a recording in the Ring app.

### How it works - the gory details
The concept for streaming is actually very simple and credit for the original idea must go to gilliginsisland's post on the [ring-hassio project](https://github.com/jeroenterheerdt/ring-hassio/issues/51).  While I was already working on a concept implementation for ring-mqtt, when I read that post I realized that it was actually a strong model that could be married with ring-mqtt to support live streaming in a way that made a lot of sense.  The post described a method to use rtsp-simple-server and its ability to run a script on demand to directly run a node instance that leveraged ring-client-api to connect to Ring and start the live stream.  Since ring-mqtt already ran in node and used ring-client-api, and already contained code for starting streams, I decided that instead of starting a separate node script, which had high startup delay and memory overhead, I could just have rtsp-simple-server run a simple shell script that used MQTT as the lightweight control channel to signal ring-mqtt to start/stop the stream on demand from streaming clients while also allowing streams/recording to be easily controlled manually via MQTT commands, which was also a commonly requested feature.

Digging more into rtsp-simple-server, I found that it not only had the ability to run a script on demand, but also included a simple REST API so that it could be easily configured and controlled dynamically from the exiting ring-mqtt node instance.  Below is the general workflow:

1) During startup, ring-mqtt checks if camera support is enabled and, if so, spawns and monitors an rtsp-simple-server process.
2) After device discovery, ring-mqtt leverages the rtsp-simple-server REST API to register the RTSP paths with a configured on-demand script handler that will execute the shell script and pass the proper MQTT topic information for each path using environment variables.
3a) A stream is started by any media client connecting to the RTSP path for the camera (rtsp://hostname:8554/<camera_id>_live or hostname:8554/<camera_id>_event).
-- or --
3b) The command to manually start a stream is received via MQTT.  In this case ring-mqtt internally starts a small FFmpeg process that connects as an RTSP client to trigger the on-demand stream.  This process does nothing but copy the audio stream to null, just enough to keep the stream alive while keeping CPU usage extremely low.  
4a) If an existing stream is already active and publishing to the requested path, the client simply connects to the existing stream in progress
-- or --
4b) If there is not an active publisher for that stream, rtsp-simple-server runs the shell script to start and monitor the progress of the stream via MQTT.  Since this is just a simple script sending a single MQTT command to an already running process, the startup process is fast and lightweight as ring-mqtt already has a connection to the Ring API and the communication channel with MQTT is local and takes only a few ms.  Usually, the live stream starts in ~2-3 second although buffering by the media client usually means the stream takes a few additional seconds to actually appear in the UI or media client.  
5) The RTSP server continues to stream to the clients, once the stream times out on the Ring side, or the last client disconnects, rtsp-simple-server stops the on-demand script, which sends the MQTT command to stop the stream prior to exiting.

The overall process is fairly light on CPU and memory because the ffmpeg process that is receiving the stream is only copying the existing AVC video stream to the RTSP server with no modification/transcoding.  The only transcoding is of the audio stream because the primary stream is G.711 μ-law while the Home Assistant stream component is compatible with AAC so the FFmpeg process does create and stream a second AAC based audio channel for maximum compatibility (players can choose the stream with which they are compatible).
