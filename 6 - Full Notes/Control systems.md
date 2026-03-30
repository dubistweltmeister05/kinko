[[Twitter Posting]]


Right, let me put you guys on.

This right here, is a diagram of a Linear control system called a PID control system. It stands for Proportional, Integral, and Derivative control. And trust me when I say this - this is one of the MOST fundamental concepts of the modern world.

There isn't a mechanical system that can function correctly without a PID loop. Excusing my hyperbole for a moment, let me explain.

A system, quite simply put, is an abstract object that accepts inputs and produces outputs in response. Systems are often composed of smaller components that are interconnected together, leading to behavior that is more than just the sum of its parts. In the control literature, systems are also commonly referred to as plants or processes.

The most relatable system that I can give you an example of would be that of a Car. The mechanical system takes the accelerator, break, and the steering wheel as it's input, and produces torque and rotates the wheels in response to the inputs that it received. Of course, the end goal and the outcome of this exercise being, well, locomotion.  

Now, what remains to be understood, is that can we figure out 2 things -

1 - Can I ensure that the output shall be perfectly replicable every time I enter the same input.

2 - Can I predict the inputs that are needed to achieve a particular output.

  

The answer to the first one, is YES, but only in an ideal world. In the unfortunate reality that we co-exist in, however, it isn't the case. There is almost certainly a component that is un-accounted for that comes in play - we term this as disturbance, or noise. This is something that plays an important part in control theory - I would even say that this is the reason for the existence of control systems and theory in the first place!

Noise, or error is something that I cannot account for. I don't know the magnitude of it, and neither can I predict the same. Thus, I need to react to it. And the methods of reacting to said noise, so that the outcome of my system remains the same, is the whole point of control theory.

And as far as the second question is concerned - the answer is yes, we can. Depending on the method that we choose to try and achieve this, the type of control system that we come up with shall vary. The 2 most basic ones, are Feed-Forward control systems, and Feed-Back control systems. Let's take a look - 

### Feed Forward Control systems
![[Screenshot 2026-03-22 084941.png]]

A feedforward control system is a type of control strategy where the control action is determined solely based on the input and a predefined model of the system, without relying on feedback from the output. Instead of reacting to errors after they occur, the controller anticipates the required action by predicting how the system should behave for a given input.

In such systems, disturbances can also be accounted for in advance if they are measurable. The controller uses knowledge of the system dynamics and expected disturbances to compute the appropriate control signal, aiming to produce the desired output directly. This makes feedforward control inherently proactive rather than reactive.

However, because it does not measure or correct the actual output, its performance heavily depends on the accuracy of the system model. Any mismatch between the model and the real system, or the presence of unmeasured disturbances, can lead to errors that the system cannot correct on its own.

However, it is evident to anyone who is still reading this so far that it isn't the best method that we have come up with, is it! What if, in addition to the inputs that we are already providing to the system, we could also let the control know the PRESENT STATE of the output, so that it can tweak the system input in such a manner that the FUTURE STATE of the output aligns with what I want it to be?

Well that, is called as -

### Feed Back Control System

![[Screenshot 2026-03-22 090201.png]]

Here, we indeed care about the current, or Present state of the output, and add that as an Input to the controller. The controller already knows the target output state, also called the setpoint that the system is supposed to achieve, and thus, it calculated the ERROR between the current and the target state. The whole game here, is to minimize that error, by moving the system in a suitable manner. 


Now, there are MULTIPLE ways of getting that done. We, are going to focus on the simplest one - liner PID Conrolers. 

### PID Control
![[Screenshot 2026-03-22 093615.png]]

Look at this beautiful bastard here man. Isn't this marvelous? Anyways, I am being sidetracked. 

A PID controller is something that utilizes a closed-loop feedback control mechanism that continuously adjusts outputs based on the difference between a desired setpoint and the measured value.

The core principle is quite easy to understand, there are 3 control units of the system, the Proportional, the Integral, and the Derivative control units. Each of these units modifies the value of the output in a unique manner. 

The **Proportional (P)** term responds to the present error, generating an output proportional to its magnitude. By applying immediate corrective action, the P term minimizes errors quickly.

The **Integral (I)** term addresses any persistent errors or long-term deviations from the setpoint by accumulating the error over time. By integrating the error signal, the I term ensures that the system approaches and maintains the setpoint accurately, eliminating steady-state errors.

The **Derivative (D)** term anticipates future changes in the error by evaluating its _rate of change_. This approach dampens oscillations and stabilizes the system, especially during transient responses.

So, the controller is made aware of the present state of the output via the feedback mechanism. It then continuously calculates the error between the setpoint and the current state of the system. Via the (P) unit, it generates a control signal that is proportional to the magnitude of error. Via the (I) unit, it sums up all past errors and generates a control signal that ensures a steady-state response to a step input to the system. As the name suggests, it is calculated as the integral of the error over time. Finally, via the (D) unit, the rate of change of the error is dampened, so that the system is not stuck in a perpetual loop of over and undershooting as it approaches the setpoint. 

Of course, there has to be a way to control the impact of all these units of control, and that is done via the Gain Constants of the controller, creatively named Kp, Ki, and Kd. The main game that is played when we design a PID controller is to fine-tune these gain constants to the system that we are modelling for. Adjusting the values helps us achieve the desired control over our system.

Trust me, it is NOT as easy is it sounds. People really make a career out of this! And while the methods of tuning are out of the scope of this article (translation - this is already long enough for you tik-tok brained bastards), I'll leave you with the names for you to go explore - Ziegler-Nichols Method, and algorithms like gradient descent, genetic algorithms, or particle swarm.

If you've read this so far - first of all, thank you so much. I have started writing this at 7:40 AM and I finished this at 8:08AM, just before I start getting ready for work. Do share this if you found it useful, and follow me for more of such articles!