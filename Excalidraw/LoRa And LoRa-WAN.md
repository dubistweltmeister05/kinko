[[RhyGen]]
[[Twitter Posting]]
[[Blog Topics]]

# LoRa
Long Range, is an insanely creative choice for naming a comms protocol, lol. 

It is developed by SemTech, and is specifically made for communicating data at insanely long ranges of distance while using extremely low amounts of power. We are talking about a range around 10-20KM in rural areas, and the **SX126x** series consumes **4.6 mA** during reception at 125 kHz bandwidth and **58 mA** during transmission at 17 dBm output power. It's a marvel of tech that is quite new so to speak, being  developed in 2009, and the first modules for the same being released near 2013.

The magic that LoRa has lies in something called CSS - pipe down web developers, this one is called Chirp Spread Spectrum.  The chirp signal is a sinusoidal signal who's frequency increases or decreases over time (often with a polynomial expression for the relationship between time and frequency). Spread Spectrum is a technique wherein by which a signal (e.g., an electrical, electromagnetic, or acoustic) generated with a particular bandwidth is deliberately spread in the frequency domain over a wider frequency band. Spread-spectrum techniques are used for the establishment of secure communications, increasing resistance to natural interference, noise, and jamming, to prevent detection, to limit power flux density (e.g., in satellite downlinks), and to enable multiple-access communications.

Now, how does LoRa use CSS signals to it's advantage? 

## CSS Operation, Spreading Factor and Other Parameters

LoRa transmits information using chirps that sweep across a predefined frequency range. Rather than representing data using fixed frequencies or phase shifts, each symbol is encoded as a chirp with a specific cyclic shift. The receiver correlates incoming chirps with locally generated reference chirps to determine the transmitted symbol. This correlation process is one of the reasons LoRa receivers can successfully decode signals that are significantly weaker than the surrounding noise floor.

One of the most important parameters governing LoRa communication is the **Spreading Factor (SF)**. The spreading factor determines how many chirp states can be represented within a symbol and directly affects both communication range and data rate. A higher spreading factor increases the amount of information encoded into each chirp and provides greater processing gain at the receiver. This makes the signal easier to detect over long distances and in noisy environments. The tradeoff is that each symbol takes longer to transmit, reducing the overall data rate and increasing packet airtime.

Closely related to the spreading factor is the **bandwidth (BW)** of the transmission. Bandwidth determines the frequency range over which each chirp sweeps. Wider bandwidths enable higher data rates because the chirps complete their frequency sweeps more quickly. Narrower bandwidths improve receiver sensitivity and communication range at the expense of throughput. Most LoRa deployments use a bandwidth of 125 kHz, which provides a practical balance between performance and range.

Another important parameter is the **coding rate (CR)**. LoRa incorporates Forward Error Correction (FEC) to improve reliability in the presence of interference or weak signal conditions. Additional redundancy can be added to transmitted packets, allowing the receiver to recover data even when portions of the transmission are corrupted. Higher coding rates improve robustness but reduce the effective payload throughput.

The interplay between spreading factor, bandwidth, and coding rate allows LoRa networks to be tuned for a wide variety of applications. A battery-powered sensor transmitting a few bytes every hour may prioritize maximum range and reliability by using a high spreading factor and significant error correction. Conversely, a device located close to a gateway may use a lower spreading factor and wider bandwidth to minimize airtime and increase network capacity.

The extraordinary range achieved by LoRa is not the result of unusually high transmit power. Instead, it is a consequence of the processing gain provided by Chirp Spread Spectrum modulation, combined with highly sensitive receivers and robust error correction mechanisms. This allows LoRa receivers to decode signals that would be indistinguishable from noise to many conventional radio systems, making long-range communication possible even with modest transmit powers.

---

## LoRaWAN – The Protocol Built on Top

While LoRa defines how information is transmitted over the air, it does not define how devices are identified, authenticated, secured, or managed within a network. These responsibilities are handled by **LoRaWAN**, a networking protocol developed by the LoRa Alliance specifically for large-scale deployments of LoRa devices.

A LoRaWAN network consists of end devices, gateways, network servers, and application servers. End devices communicate with one or more gateways using LoRa radio transmissions. The gateways act primarily as packet forwarders, relaying received data to a network server through a conventional IP-based connection such as Ethernet, Wi-Fi, or cellular networks. The network server is responsible for packet deduplication, security checks, device management, and routing data to the appropriate application.

One of the primary advantages of LoRaWAN is its security model. Communications are protected using AES-128 encryption, and separate keys are maintained for network-level and application-level security. This architecture ensures that devices can be authenticated and data can remain protected even when transmitted through third-party gateways and infrastructure.

LoRaWAN also introduces the concept of device classes, allowing applications to balance responsiveness against power consumption. Class A devices, which are the most common, operate with extremely low power consumption and only listen for downlink messages immediately after transmitting. Class B devices periodically open additional receive windows according to a synchronized schedule, enabling more predictable downlink communication. Class C devices remain in a near-continuous receive state and provide the lowest communication latency, albeit with substantially higher power requirements.

To improve network efficiency, LoRaWAN supports **Adaptive Data Rate (ADR)**. ADR allows the network server to dynamically adjust parameters such as spreading factor and transmission power based on link quality. Devices close to a gateway can therefore transmit more quickly and occupy the channel for shorter periods, while distant devices can be configured for greater robustness. This improves the overall capacity and scalability of the network.

The distinction between LoRa and LoRaWAN is important. LoRa is simply the radio technology responsible for moving bits through the air. LoRaWAN builds upon that foundation to provide addressing, security, network management, and large-scale deployment capabilities. In practical terms, LoRa enables communication between radios, while LoRaWAN enables the creation of secure, scalable networks containing thousands of devices.

