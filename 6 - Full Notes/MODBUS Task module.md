[[RhyGen]]

## Context of the operation
We have to talk to only a single device for now, and we know exactly what we need to poll on the MFM. The slave ID shall be 0x01, and the underlying UART baud rate is 115200. I was earlier for a bunch of values that are spread out over a long interval - the voltages and currents being at the very beginning of the modbus map, and the power values being quite later. Now, my requirements boil down to simply caring about the power values of the MFM. Looking at the register map, the addresses to be checked are from 40043 to 40057. 

## Improvements over the FSM Architecture. 
I want to rework this into 2 simple operations. One tx operation, that requests data from the MFM over MODBUS for the address ranges that I have specified, and an Rx operation that polls a UART ring-buffer, which is used to record the response to the request from the MFM. Ideally, I think that this should work, since I explicitly have a single device that I am talking to over MODBUS, that there is significantly lower number of registers that I am now polling for from it. 

The pattern I am looking to follow is along what I see in the app_tx_ecu_request() and the handler that takes care of the response to it - handle_ecu_resp. The structuring of the tx and the handler should be maintained in the app_msgs for my DG application. 