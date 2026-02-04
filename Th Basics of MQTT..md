
## Defining MQTT
A publisher/subscriber model of comms. It is lightweight, open, simple, and designed so as to be easy to implement. These characteristics make it ideal for use in many situations, including constrained environments such as for communication in Machine to Machine (M2M) and Internet of Things (IoT) contexts where a small code footprint is required and/or network bandwidth is at a premium.

## Participants 
There's 2, a broker and a client. The MQTT Pubs are capable of sending multiple values over channels called "topics" that help differentiate between the output values. The subscribers, can pick topics of their interest and choose to ignore the other values recieved. 

To facilitate this, we have a Broker, which maintains a list of the subs that have subscribed to a particular topic and is responsible for sending the relevant data to each one of these. 