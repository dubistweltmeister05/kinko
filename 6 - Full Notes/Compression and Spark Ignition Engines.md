[[Twitter Posting]]
[[Blog Topics]]

For anyone trying to understand internal combustion engines, the most important distinction is not the fuel itself, but the way combustion begins inside the cylinder. A spark ignition engine and a compression ignition engine both take air, squeeze it, and burn fuel, but they use fundamentally different physics to start that burn. This article expands that comparison into readable paragraphs, with enough technical detail to make the difference clear.

# CI and SI Engines: A Detailed Comparison
CI (Compression Ignition) and SI (Spark Ignition) are the two principal combustion systems in internal combustion engines. In an SI engine, a premixed air-fuel charge is ignited by a spark plug. In a CI engine, only air is compressed and fuel is injected into the hot air, where it autoignites. That distinction changes everything from combustion timing and fuel system design to efficiency and emissions.

Most passenger car gasoline engines are SI engines, while most diesel engines are CI engines. But the real contrast lies in mixture preparation, ignition control, and the way the engine handles pressure and temperature during the cycle.

---
## Historical Context
The spark ignition engine is historically linked to Nikolaus Otto and the Otto cycle. Otto demonstrated in the 1870s that a four-stroke engine could be made practical by using a timed spark to ignite a pre-mixed air-fuel charge. The compression ignition engine is linked to Rudolf Diesel, whose engine used heat from compression rather than a spark to ignite the fuel.

Both systems share the same four basic steps: draw air into the cylinder, compress it, burn fuel, and convert the expanding gases into mechanical work. Where they differ is how the mixture is formed and how combustion is triggered. This difference leads to different mechanical requirements, different efficiency limits, and different use cases.

---
## Spark Ignition (SI) Engines
In an SI engine, the air and fuel are mixed before entering the combustion chamber or inside the intake port. That mixture is compressed and then ignited by a spark plug very near top dead center. The engine relies on controlled flame propagation, and the chemistry of the fuel determines how much compression can be used before the mixture detonates unintentionally.

### Fuels used in SI engines
Gasoline is the most common fuel, but a spark ignition engine can also run on ethanol blends, methanol, CNG, LPG, and even hydrogen in specially designed systems. The key requirement is that the fuel must be sufficiently volatile and stable to form a homogeneous charge, and it must resist autoignition until the spark is delivered.

---
## Four-Stroke SI Cycle
A modern SI engine typically runs on a four-stroke Otto cycle. The sequence is intake, compression, combustion/power, and exhaust.

### 1. Intake stroke
During the intake stroke, the intake valve opens and the piston moves down. Fresh air and fuel enter the cylinder. In a carbureted engine, the fuel is mixed with the air in the intake manifold. In injected engines, fuel is sprayed either into the intake port or directly into the cylinder. The goal is a well-mixed, homogeneous charge before compression.

A typical stoichiometric air-fuel ratio for gasoline is about 14.7:1 by mass, meaning there is 14.7 times more air than fuel. That ratio provides a balance between complete combustion, acceptable power, and reasonable emissions.

### 2. Compression stroke
Once the intake valve closes, the piston rises and compresses the air-fuel mixture. In a conventional gasoline engine, compression ratios range from about 8:1 to 12:1. Modern engines with direct injection and knock management can go higher, often 13:1 or 14:1.

Compression increases temperature and pressure, shrinking the mixture into a smaller space and making it easier to ignite. However, too much compression pushes the mixture toward autoignition, which causes knock. That is why gasoline engines are limited by the fuel’s octane rating.

### 3. Power stroke
At or near top dead center, the spark plug fires and creates a small flame kernel. That kernel grows and the flame front travels through the combustion chamber, consuming the mixture. Combustion is not instantaneous; the flame moves across the chamber in a controlled front.

Because the pressure rises while the piston is still near top dead center, the expanding gases do useful work pushing the piston down. This power stroke is the only stroke in the cycle that produces net mechanical output.

### 4. Exhaust stroke
After combustion, the exhaust valve opens and the piston moves upward. The burned gases are pushed out of the cylinder through the exhaust port. With the chamber emptied, the cycle begins again.

---
## Otto Cycle and Efficiency
The idealized SI engine follows the Otto cycle. In a simplified model, thermal efficiency depends on the compression ratio and the specific heat ratio of the working gas. Higher compression ratios improve efficiency, which is why gasoline engines with high octane fuel can be more efficient than lower-compression designs.

Because real engines also have losses from pumping, friction, and heat transfer, practical thermal efficiency is lower than the theoretical value. Typical gasoline engines convert about 25% to 40% of the fuel’s energy into useful work.

---
## SI Combustion Process
SI combustion consists of several distinct phases:

- Ignition delay, where the spark is generated and the flame kernel forms.
- Flame development, where the kernel reaches a size capable of sustaining combustion.
- Flame propagation, where the flame front moves through the chamber.
- After-burning, where remaining unburned gases combust.

Flame speed matters a lot. Faster flame speed means more of the charge burns at a favorable crank angle, improving efficiency and reducing emissions. Slower combustion increases the chance of incomplete burning and can reduce performance.

### Knock in SI engines
Knock occurs when the unburned gas ahead of the flame front autoignites spontaneously before the front arrives. This creates pressure waves and a rapid pressure rise instead of a smooth burn. Knock causes noise, vibration, reduced power, and can damage pistons, rods, and bearings if allowed to continue.

The engine’s susceptibility to knock depends on compression ratio, intake temperature, mixture strength, and fuel chemistry. Engineers manage knock with spark timing, exhaust gas recirculation, charge cooling, and fuel selection.

### Octane number
The octane rating of gasoline measures its resistance to knock. Higher octane fuel resists autoignition and allows the engine to run at higher compression ratios or with more aggressive spark timing. Regular gasoline is usually around 87 AKI, while premium grades can be 91 to 93 AKI or higher. High-performance engines often require premium fuel to avoid detonation.

---
## Fuel systems in SI engines
The way the air-fuel mixture is delivered has evolved significantly.

### Carburetors
Early engines used carburetors, which rely on airflow through a venturi to draw in fuel. Carburetors are simple and inexpensive, but they provide poor control over mixture ratio, especially during transient conditions, cold starts, and varying altitudes. This leads to higher emissions and reduced efficiency compared to injection systems.

### Port fuel injection
Port fuel injection sprays fuel into the intake port ahead of the intake valve. This gives better dispersion than a carburetor and allows electronic control over fuel quantity and timing. Port injection improves fuel economy, lowers emissions, and supports more precise air-fuel control.

### Gasoline direct injection
Gasoline direct injection sprays fuel directly into the combustion chamber. This enables better atomization, more precise charge preparation, and stratified-charge operation in some engines. Direct injection also supports higher effective compression ratios because the cold fuel spray helps reduce knock tendency during compression.

---
## SI engine characteristics
Spark ignition engines generally excel at high engine speeds and smooth operation. They are typically lighter and smaller than diesel engines of similar power, allowing higher power-to-weight ratios and good transient response. They also tend to have lower vibration because the combustion event is more gradual and the engine structure can be optimized for lighter loads.

However, SI engines are limited by knock and throttling losses. At part load, power is controlled by restricting airflow with a throttle body, which creates pumping losses as the engine draws air through a pressure drop. That reduces efficiency compared to engines that can regulate power by fuel quantity alone.

Typical modern gasoline engines achieve 25% to 40% brake thermal efficiency. High-performance engines with direct injection and variable valve timing can approach the upper end of that range, but they are still usually less efficient than a well-tuned diesel.

---
# Compression Ignition (CI) Engines
Compression ignition engines take a different approach. Only air is drawn into the cylinder and compressed. Diesel fuel is injected near the end of compression, and the hot, compressed air is sufficiently energetic to ignite the fuel without a spark plug.

Because combustion is triggered by autoignition, CI engines are sometimes called diesel engines. But the key is that compression is used to produce the ignition temperature, not a spark.

---
## Four-stroke CI cycle
The CI engine also follows a four-stroke process, but the details are different.

### 1. Intake stroke
During the intake stroke, only clean air enters the cylinder. There is no fuel in the intake air because diesel combustion requires a lean, oxygen-rich environment and the fuel must be injected precisely into the hot air near top dead center.

### 2. Compression stroke
The piston compresses the air alone. Diesel engines use much higher compression ratios than gasoline engines, typically between 14:1 and 24:1. The compression heats the air to temperatures in the range of 600°C to 900°C, which is enough to ignite diesel fuel when it is sprayed into the chamber.

The higher compression ratio yields higher thermal efficiency and also means the engine can use a fuel with a lower autoignition threshold than gasoline. But it also requires a stronger, heavier engine structure to handle the greater pressures.

### 3. Fuel injection
Near the end of the compression stroke, diesel fuel is injected into the hot air at extremely high pressure. Modern common-rail diesel systems often operate at 1,500 to 3,000 bar or more. High pressure produces very fine droplets, which vaporize quickly and mix with the hot air.

The injected spray must be carefully shaped. The nozzle design, spray angle, penetration depth, and timing all influence how the fuel mixes and how the combustion process evolves.

### 4. Auto-ignition and power stroke
Once the fuel has vaporized and mixed with hot air, it begins to autoignite. CI combustion is not a single, instantaneous event. It has phases: ignition delay, premixed burn, mixing-controlled combustion, and late combustion. During the power stroke, pressure rises as the fuel burns, and the expanding gases push the piston down.

Because more fuel is still being injected while combustion begins, the engine can sustain power output by controlling the rate of fuel injection rather than by throttling air. This is one reason diesel engines are efficient at part load.

### 5. Exhaust stroke
After the cylinder pressure drops, the exhaust valve opens and the piston expels the burned gases. The cylinder is then ready for a fresh charge of air.

---
## Diesel Cycle and efficiency
The idealized CI engine is described by the Diesel cycle. In this model, efficiency depends on both compression ratio and the cutoff ratio, which is the fraction of the cycle during which fuel is injected and burned.

While a Diesel cycle at the same compression ratio is theoretically less efficient than an Otto cycle, a real diesel engine operates at a much higher compression ratio. That makes modern diesel engines more efficient in practice, especially at part load and for heavy-duty applications.

---
## CI combustion phases
Diesel combustion involves distinct phases:

- Ignition delay: the time between injection and the start of combustion. During this period fuel atomizes, vaporizes, and mixes with air.
- Premixed combustion: the portion of fuel that is already vaporized and mixed ignites rapidly, creating a sharp rise in pressure.
- Mixing-controlled combustion: as more fuel is injected, the burn rate is limited by how quickly it mixes with air.
- Late combustion: remaining fuel finishes burning, and the pressure begins to fall.

The ignition delay and premixed burn are the main reasons diesel engines produce their characteristic noise. Engineers reduce that noise by using pilot injection, multiple injection events, and advanced injection control.

### Cetane number
Cetane number is the diesel equivalent of octane rating. It measures how quickly a fuel autoignites under compression. Higher cetane fuels produce shorter ignition delays, smoother starts, reduced combustion noise, and better cold-weather performance.

---
## Importance of fuel injection
Diesel engine performance depends heavily on injection quality. The size of droplets, spray pattern, injection timing, and pressure all determine how well the fuel mixes with air and how cleanly it burns.

Modern common-rail injection systems allow extremely precise control. They can perform pilot injections before the main event and post injections afterward. This improves efficiency, reduces soot and NOx emissions, and makes the engine quieter.

---
## CI engine characteristics
Compression ignition engines are typically more efficient than spark ignition engines. Brake thermal efficiency for diesel engines can range from 35% to 50%, and large marine or stationary diesels can exceed 50%. The higher efficiency comes from higher compression ratios, lower throttling losses, and a combustion process that is controlled by fuel quantity.

Diesel engines also produce high torque at low speeds, which is why they are widely used in trucks, buses, and heavy equipment. Their construction is heavier and more robust because they must safely contain higher peak pressures.

The downsides are lower maximum RPM, more vibration, and a more expensive fuel system. High-pressure pumps and precision injectors add cost, and diesel combustion can produce more particulate matter and nitrogen oxides unless the exhaust system includes advanced aftertreatment.

---
## Comparing SI and CI engines
The fundamental trade-offs are:

- SI engines are lighter, smoother, and better suited to high-revving applications.
- CI engines are more efficient, produce more torque, and are better suited to heavy-duty, long-duration operation.

SI engines rely on spark timing and charge quality, while CI engines rely on compression heating and injection control. Understanding this difference helps explain why gasoline engines dominate cars and motorcycles, while diesel engines dominate trucks, ships, and stationary power.

If you want a deeper takeaway: the combustion method dictates the entire engine architecture, from fuel delivery to cylinder pressures to exhaust aftertreatment. Once you see that, the rest of the engine design decisions fall into place.

### Knock Limitations

Compression ratio is limited by knock resistance.

---

