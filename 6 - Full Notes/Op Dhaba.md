[[RhyGen]]

## Things that I am in charge of
- Firmware Dev and Test For Logger 0.2
- Cloud Server and Infra For Logger 0.2
- Socials

# Plans for each

## Firmware Dev and Test For Logger 0.2
Here is all that needs to be developed, either from scratch or from some prior dev -
- RTC
- SMS
- GPS
- IMU
- FOTA
- FRAM

Trello cards with details for each shall be made and maintained with details and tracking of progress at every EoD. Testing procedures for the logger firmware so far and the cloud infra that's associated with it needs to be designed and implemented soon enough. Think of making feature tests and bring-up planning for all of these. A testing branch with macros to enable and disable testing routines in a single application shall be a nice way to get this done. 
## Cloud Server and Infra For Logger 0.2
Need to talk to consultant soon and get his plan finalized. Once it looks good, STUDY ANY AND EVERYTHING THAT'S PUT IN THERE. Optimization and monitoring shall be of paramount importance. Save costs if you can, but it's okay if you cannot for now. The first goal is to get a reliable and stable state on the cloud Infra. 

I need to finalize 
- the storage scheme 
- the instance type
- the format in which we are storing our logs 
- the multi-zone backups
- the dashboard structure
- the operations that we wish to run on the collected data. 

As I keep taking care of this, I also need to have methods to ingest the Ruptela and teltonika data that is being sent for me. So, copy paste all those points for that data too. Start with understanding how can we configure the data that is being sent from these loggers. 

Another suggestion for these shall be that we only deploy custom loggers in DG sites. And Ruptela or Teltonika loggers in the vehicle / bibtya. 

## Socials
Play the game as you know it to be the best. The only thing that matters is numbers and awareness about us and what we are doing. Launch, publicity, visibility and perception of us and our work is of the utmost importance. 

Track these things and the numbers that they generate across all platforms. Not just twitter but everything else. 
