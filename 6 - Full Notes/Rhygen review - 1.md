[[RhyGen]]

# outcomes achieved
## Logger 0.1
Functional application for DGs, with features for capturing engine parameters via CAN over J1939, Electrical Parameters from MFM via MODBUS over RS485, and SD card storage of logs in binary format. A functional driver and interfacing application for A quad-SPI based memory module was also achieved alongside the project. 

Establishing communication with Quectel's EC-200 4G module, and setting up an MQTT Pipeline with a basic test of wireless logging of measured parameters. 

## Zest Project
Complete system design for feeder boards, master logger and HMI endpoint.
Finalization of field sensors that need to be procured from Zest.
Negotiation of defined deliverables, pricing and delivery timeline. 
Quotation of prices for our systems and MoU finalization. 
## Bibtya HMI
Developed and tested an HMI system, integrating a TFT display and CAN communication within a bare-metal touchGFX project, along with application code for Interfacing an external joystick with the SCU, and publishing messages over the CAN bus interface. 
# things stalled or fell short
## Zest
We are essentially dealing with technology brokers, with our role in the project being dependent on inputs from parties that we have no contact with. Zest is broking that information for us, with significant delays, that ended up cascading over - leading to the project being delayed by about 3 months. 

There have also been delays from me with this process, due to my own initial lack of significant interest in dealing with exetrnal entities and going back and forth with them. I found myself hesitant to ask questions, delaying conversations with them, and not following up with them as aggressively as I could have or rather - should have. These issues, have compounded to delay the signing-off for the project by about 2.5-3 weeks, in my own estimation. 
## Tor Support and Logger Deployment
While working on interfacing with the Tor MFM, we found that the meter's data update rate via Modubs over RS485, is nearly 800ms, while their promised internal sampling rate was supposed to be 80ms. We tried to raise this issue with Tor through multiple channels to get this sorted out, but to no avail. 

I started with personally contacting my old boss at Tor directly, and the engineer who works on it himself. My old boss got on a call and promised to resolve this internally with their own team, but that did not result in any significant progress. 

We then contacted the support channel there, and got a confirmation that while the change we needed is technically possible, they are unwilling to do so, for it does not match with their internal product development roadmap. 

Finally, we contacted their head of business and head of sales, to make a business case for the change that we wish to have done for us, but were told that the potential business that shall come form us is not commercially worth their time to accommodate our change.

Lastly, I contacted my friend working at Tor for access to compiled binaries of the titan firmware, and I received the latest version. But that proves to be futile, since it is equipped with the dummy device ID, and flashing that over on the MCU we have is not of any use for us. I also happen to maintain an old version of the project's firmware, and while it is technically possible for me to make the change in the old project and flash the meter, I am afraid of bricking the MFM - hence the reluctance to do so. 

I even tried to convince my friend in order to gain access to the current version of the source firmware, but this goes strongly into the illegal territory, and I am hesitant about proceeding with this route, since I would be solely liable and responsible of any consequence that shall befall us in the future.   
## HMI Deployment
There were changes in the nature of the controlling joystick - switching from a digital button based module to a potentiometer based analog module, and the signal routing of the control input that were incorporated quite late into the project, leading to a few significant re-writes of the developed firmware, both for the HMI's MCU and the SCU's interface with the joystick. 

For the development of the Hardware circuitry, more prior tasks, such as the FEV test and other work related to the vehicle for sarthak, led to significant delays. 

As of now, the development stands complete and is ready to be installed. 
# learnings and experience

## Overwhelmed by the scope of work
I came in, expecting that my job as a firmware engineer is limited to developing embedded software, testing it on the system at hand, and being done once the firmware works. I soon found out how wrong I was, and it took me a while to come to terms with what is expected of me to do, and at what extent am I expected to be involved. 

I have been slowly wrapping my head around it and been trying to do better, but I admit that a lot better needs to be done than what has been executed so far. Be it my involvement with managing the Zest project, or dealing with people over at Tor - and my own hesitations in pestering the people I used to work with. 

And when deadlines are missed, or things don't go as they are expected to, I find myself spiraling into a loop of self-blame. Frankly speaking, I am dead-scared of losing my employment for a lot of personal reasons, and when combined with things not happening - it lead to a constant cloud of self-doubt and fear in my heads-pace, causing  bigger dent in my productivity.

The way of fixing my shortcomings when it comes to productivity at work 
## Ownership of written and committed code. 
After being accustomed to completely LLM driven development cycles at my previous employment, I had gotten used to not owning the code that is committed, and generating outcomes as quick as possible for pleasing the team leads and management. Since development cycles were mostly in-house, I never faced any customer facing deployments and thus, became way too comfortable with not owning what I develop - as long as the intended outcome was achieved. If it is to design a screen, then all is good as long as the data is displayed, and the transitions occur as they should. If it was to design an alarm system, just triggering the alarms via some simulated data was enough to park the task. 

That attitude carried over when I joined, and the effects resurfaced during a review with Sajal. I was rightfully called out, and I was given the grace to come up with a solution and fix this issue. Since being pointed out, I have aggressively reduced my dependency on LLMs for designing systems, and have come up with my own ideas and personal habits for using LLMs more effectively, allowing me to ship code that I truly understand and own. An opportunity implement these fixes first came up with the HMI developement, and now presents itself through the continuation of my work on the logger and it's planned deployment. 

## Seeing tasks through till the very end
Coming from corporate, I have been only concerning myself with what I am assigned to do. If it's firmware, then I concerned myself only with development and testing of said firmware on the MCU, and it's related sensor interfacing. I would simply park the tasks and let someone else worry about what is to be done beyond my work. 

I quickly found out that this attitude is not going to work out here, and something needs to be done to fix it. Again, this was pointed out by Sajal, hwo had a long conversation about this with me after one of our weekly team meetings.  

I cannot say that I have completely fixed this flaw of mine, since I still miss out on multiple aspects that are involved in the process of deploying our products - and I attribute this to it being the first time I have faced responsibilities as such. The only way to fix this that I see, is to continuously think about what more needs to be done with the products that I am involved with (eventually, with all products and projects that go on at Rhygen), make sure that they are recorded in some written way, and to keep on iterating over it, completing things till the end.

As I see it moving forward, - continued work on the Zest Project and logger deployments are good opportunities for me to fix up and achieve better and faster outcomes.
## Frequent Leaves from the office
No excuses here - I have been missing time from work, for personal reasons such as injury or health concerns, that I need to do a better job at keeping in check. A major change that is due from 3rd October, which I think shall help with this is moving much closer to work. 

I believe that cutting down my travel time from 45mins either way, to barely 7 minutes shall play a significant part in me being more present, mentally and physically, at our place of work, allowing me to be a lot more productive with what I am being assigned responsibility of, and then going beyond as need arises.   

# Feedback 
for business, engineering, vision, office setup, founders, coworkers, culture, management style

A more conversational approach of management would work better in my view. A small update by the team leads from their team members, about what was done in the day, what moved forward ( and maybe what could be done better)  as they leave for the day would serve as self-reflection for all of us. I personally find myself mentally spent as the day draws close, and an hour long drive through the Pune traffic is not exactly helping me recharge my energy - leaving little to no time for self-reflection about the state of my work. Of course, moving closer to work shall help, and so shall a short 5-minute conversation with my team lead as I am about to depart for the day. 


Overall - Being at Rhygen is a chance for me to become a better engineer, through the work that I get to do here. It is a privilege to be overwhelmed with a lot to do, and I need to do a better job at earning a right to that privilege. The failure of me not being to move the needle far enough with work that I am responsible for is not lost on me, and I recognize that I need to do better as an engineer and be better as a teammate during my time here.  That's all I have to say for myself.



# TODOs after the meet
1. Make project timeline and identify bottlenecks in the processes.