[[RhyGen]]

Talk a bit about what we are developing, and then introduce how the MFM is used in our hybrid DG system - 

We need to measure the electrical load characteristics of the operation, so that we can tune our control algorithms to precisely produce the output power that is demanded. For this, it is paramount that we are able to catch all electric spikes that might arise in the load. Often, these spikes last for less than 100ms, and it is critical for our operation that we are able to measure and log them. We have had conversations with the support team of Tor, who have confirmed that a version of firmware, that publishes values over MODBUS at 80ms, instead of the ~800ms as observed is technically possible, and we request this to be implemented for our applications. 

We are fully willing to take responsibility of all validation and QA procedures, and don't need tor to do any checks on this for us. 