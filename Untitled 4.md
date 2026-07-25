
One more caveat: this is deployed in a village. So forget reliable broadband and say hello to LTE Cat 1bis.

My approach would be to make video completely on-demand.

Assume the cameras are configured for ~1080p at 4-6 FPS. The user clicks Camera #3 in the dashboard. The backend signals the site Raspberry Pi, which opens an RTSP session to Camera #3, bridges the feed to WebRTC, and streams it to the user. User switches to Camera #1? Tear down Camera #3, establish Camera #1, and continue.

Why?

Because streaming all n cameras 24/7 over a constrained cellular uplink is a great way to build a system that works perfectly in PowerPoint and nowhere else. Even though Cat 1bis is rated for up to 10 Mbps downlink and 5 Mbps uplink, real-world performance is often significantly lower. Continuous multi-camera streaming quickly becomes a bandwidth problem. A few seconds of startup latency is a far better tradeoff than pretending you can deliver multiple always-on video feeds from the middle of nowhere.

Oh, and Internally, I'd keep the cameras on a local Ethernet network and use the Pi as the video gateway.