[[RhyGen]]


## Site Visit notes 
Date -> 27/10/2026

Observations ->  
- There is a 250 KVA DG in place, for a site that draws 150-170 KVA.
- The Changeover and MSEB supply is across the street from the DG. The cables are all underground. 
- They have 2 connections from the grid and 2 changeover circuits that are manually operated . The DG output is split into 2 and supplied to each changeover circuit. If the grid goes down, they MANUALLY turn on the DG first and then switch the supply. All this is because there is no synchronization panel.

How to Deploy ->
- We can have 2 MFMs - one at each changeover output and a single logger that reads from both. It'll be at the DG, where it can read the mech params too. But - we'll need to figure out ways to get our connection wires UNDERGROUND. The only problem, is that we don't have firmware ready that can talk to multiple devices over RS485. 
- Another method could be that we deploy 2 sets of MFM+loggger - one at the end of each changeover output. Downside is that we won't be logging the DG's mech data, but we can have an earlier deployment date through this. Reconstructing the logs for true representative output is to be worked upon.
