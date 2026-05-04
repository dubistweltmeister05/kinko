[[Titan]]

DevLog - Configurable TLS Over MQTT in an ESP codebase

Right then, here is a log of the work that I have been doing at my company in the last few weeks. The task was to implement TLS over the MQTT connection that the product that I work on sends it's data over. There's 2 MQTT connections that are active on the device, a Payload server and a Parking server. To my surprise, there were literal certificates that were being hardcoded within a char array in the code, that was being flashed into the device. These were used for the parking server. And if that wasn't shocking enough - THERE WAS NO SECURE CONNECTION ON THE FUCKING PAYLOAD SERVER!

So, obviously - that has to change. Here is what I did. 

First, I created a new partition within the SPIFFS file system that exists on the device, to hold 3 files within itself - the CA cert, the Client key and the Client certification. These 3 files shall be sent via the user to their device, via a mobile app that uses Bluetooth to talk to the device. The sad part here is that bluetooth can send only 255 bytes in a single burst, and the cert and key files are about 1.5-2.4 KB each. Thus, the app breaks these certs into chunks and the device is supposed to reconstruct it as it receives. 

This would have been quite alright, IF THE APP WOULD HAVE SENT THE CHUNKS IN THE CORRECT FUCKING ORDER. Instead, I found out that the app sends a mix of chunks, each from a different certificate. Obviously, this cannot be reconstructed on the device, and causes a lot of issues. So, we raised this issue with the mobile app team, and they are working on it as we speak. 

Then, we pivoted our execution to teh following strat. For development purposes, I generated the necessary keys and certs, and burned them into the device under the special partition that we had made for the certs. Then, I made a few configuration changes to the codebase, so that it can use these certs and keys for initializing a secure MQTT (mqtts) connection to a secure swerver that we have within the company. The idea was that I can work on the parser for the incoming certificates that are to be received from the app as and when the dev team for that can sort out the chunk-mixing issues.

I focused on developing the secure connection pathway, using the generated certificates. The parser that shall be implemented, will then simply overwrite the contents of the files that are in place on the device, under the special partition that has been created. 

Of course, it works as expected. And yes, I used quite a bit of AI when I was executing it all - a decision that was forced upon the whole org by the management. Am I happy to do this? Yes and no. Yes, because it makes my job easier, and no becuase I learn a lot less when using AI. But, it is what it is at this point. There are a few improvements that need to be done, and once that's done - it's onto the next project lads. 