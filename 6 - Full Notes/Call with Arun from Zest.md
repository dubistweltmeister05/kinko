[[RhyGen]]

1 - SBC availability is an issue. We can source enough for the initial test and demo of the system at IIT- B, but the complete delivery of all 10 systems will be contingent of sourcing of the SBCs. 

2 - Alternatively, a laptop can be placed instead of an SBC, and the control algorithm of the client shall run on that laptop, with I/O interfaces being controlled via python function calls that will be written and exposed by us . These functions shall be called by their algorithm. They need to source these laptops or PCs themselves.

3 - Warranty terms need to be talked about.  From our end - 
- Prefer to ship to IIT-B, where we have a trestbench and show that it works.
- We can offer warranty for 2 months, but we cannot cover idiot-mistakes in warranty
- testing shall be done on our end before we ship, faulty boards don't reach you. 
- once a board is working and deployed, we can cover for x (2) months, but not covered accidental, water, wiring mistakes, separate component connection causing board to fail. If this happens, we will be willing to manufacture a new board for xxxx price. If the claim is proven to be genuine, we can remake the board for no additional cost to the client. 


1. **Solar digital outputs.** Since the 10 controllable digital outputs on the solar board are listed as 0 to 3.3 V, we understand these are control level signals, with the actual array row switching to be done through contactors driven via interposing relays at our end. Just for confirmation only.
   *Yes*
   *Need input on the switching voltages of their contractors, to make sure that the relay module we have in our plans can drive them as needed.*
2. **Control software on the SBC.** The Python function interface described in section 3.4 is exactly what we were looking for. Section 4.2 lists the OS image and HMI application, so it would be good if the Python I/O interface library is also mentioned there as a deliverable. Also, since the OS is described as locked down, could you let us know how we would deploy and run our own control algorithm on it, and what the update mechanism looks like in practice.
   *Can mention the interface library as well.*
   *Their control algorithm's files can be transferred to the SBC via a USB interface, and  mechanism can be set up to replace the old ones with the newer algorithm. OTA or remote updates are not being worked as of now*
3. **Section 3.4 .** The opening line still reads that the SBC is independent of the real time control system, which sits a little oddly against the heading and the points below it. Could that be reworded to match?
   *Done*
4. **Solar analog channels.** Our internal consultant noticed that table 3.2 lists 33 analog inputs in total, while two STM32F446RE MCUs would provide 32 ADC channels between them. Could you check whether the allocation covers everything listed?
   *Yes, all sensor input allocation is covered among the 32 channels of the STM32F446's ADCs*
5. **Warranty.** As discussed over the phone, it would be good if you could consider a longer warranty term, around a year rather than the 2 months currently mentioned. Do come prepared with your view on this as we agreed over phone call.
### 5.9 Hardware Warranty & Liability Terms

**5.9.1 Warranty Period & Core Scope** The Hardware Warranty applies strictly to the physical hardware units (H2 Plant Feeder Boards, Solar Plant Feeder Boards, Master Data Loggers, and SBC HMI Controllers) designed, assembled, and verified by Us.

- **Duration:** Covered by a **one (1) year Limited Manufacturing Warranty** commencing on the date of dispatch from Our facility or completion of formal bench acceptance testing at the IIT-Bombay testbench (whichever occurs first).
    
- **Scope:** Strictly covers latent manufacturing defects, PCB assembly flaws, and component non-conformance under specified operating conditions.
    
- **Pre-Dispatch Verification & Sealing:** All units undergo extensive functional and stress testing to ensure full operational compliance prior to shipment. Upon passing inspection, each hardware enclosure is secured with a physical, tamper-evident **Seal of Approval**.
    

**5.9.2 Tamper Seal Policy & Warranty Voiding** The integrity of the physical tamper seal is an absolute condition of this warranty:

- **Seal Integrity:** Any breaking, peeling, physical modification, or evidence of tampering with the enclosure seal immediately and automatically **voids all warranty coverage** for that unit.
    
- **Exceptions:** No field opening or internal modification of the enclosure is permitted without prior written authorization from Us.
    

**5.9.3 Explicit Exclusions (Field & External Hazards)** Even if an enclosure seal remains intact, the warranty **DOES NOT** cover failures resulting from external hazards, including but not limited to: a) **External Instrumentation & Wiring:** Reverse polarity, incorrect field wiring, external line-to-logic shorts, ground loops, or faults originating from third-party field sensors, actuators, or transducers. b) **Electrical Overstress (EOS):** Overvoltage spikes, lightning, electrostatic discharge (ESD) events, or power supply fluctuations beyond rated limits. c) **Environmental Exposure:** Submersion, moisture ingress through field conduit/glands, chemical exposure (including hydrogen gas degradation), or thermal conditions exceeding specified operating ranges. d) **Field Handling & Transit:** Physical impact, dropped hardware, severe vibration, or improper mounting during field deployment by the Client.

**5.9.4 Claim Procedure & Root Cause Analysis (RCA)** If an un-tampered unit exhibits an issue during the 1-year warranty period, the following procedure must be followed:

1. **Notification & Data Capture:** Within five (5) business days of an issue, the Client must submit a written report including high-resolution photographs showing the **intact enclosure seal**, terminal wiring, and applicable Master Logger logs.
    
2. **Return Shipment:** The affected unit must be packaged securely and shipped to Our facility at the Client's expense.
    
3. **Mandatory Acceptance Criteria:** Upon arrival at Our facility, the enclosure seal will be inspected first:
    
    - **Seal Intact:** We will proceed with a formal Root Cause Analysis (RCA), including microscopic trace inspection and semiconductor diagnostics.
        
    - **Seal Broken / Tampered:** The unit will be classified as void of warranty without further internal failure analysis.
        
4. **Assessment Standard:**
    
    - Within thirty (30) days of dispatch, any failure will be evaluated under standard manufacturing QC metrics.
        
    - After thirty (30) days of continuous field deployment, hardware is presumed to have been delivered fully functional. The burden of proof rests on the Client to demonstrate that an issue stems from a latent manufacturing flaw rather than external field conditions.
        

**5.9.5 Claim Resolution & Post-Warranty Hardware Replacement**

- **Valid Manufacturing Defect (Seal Intact):** We shall, at Our sole discretion, repair or remanufacture the defective hardware unit at no charge and reimburse return shipping costs.
    
- **Void Warranty / Field Failure:** If the seal is broken or the RCA reveals field-induced failure (e.g., external voltage spike entering via I/O terminals), the claim will be denied.
    
- **Absolute 1-Year Expiration & Post-Warranty Support:** Upon the expiration of the 1-year warranty period, all free technical support and warranty coverage permanently terminate.
    
- **Replacement Units:** Beyond the 1-year period (or for non-warranty field failures within the first year), hardware replacements will be provided exclusively as a paid service under a separate **Spare Hardware Agreement** (charged at BOM + assembly labor fees, subject to component availability and manufacturing lead times).






































