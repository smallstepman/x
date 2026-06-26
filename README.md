
# Observations, and notes from linked materials 

- read operation gives binary results: 
  - can either succeed with no exact information about what Vth value would've failed (although successful ECC does give us information about how many bits were corrected (RBER?)), or, 
  - fail by being unable to decode the data due to being past cliff ECC not being able to fix it, also not giving us the information about the direction in which we need to tune the parameters 
- optimal Vth is a moving target with complex dynamics (depends on temperature, charge leakage, read disturb, manufacturing variations, P/E cycles), and the drift is not a simple additive offset
  - we can precompute for manufacturing variations 
- we can gather information about previous Vth value that resulted in correct read operation (with timestamp and it's PBA to later be able to determine which neighboring cells could've been affected by the Read Disturb), and other factors like temperature
- looks like reading BER > 1.1% could be a good candidate for invoking our expensive algorithm

# Problem re-statment 

At the core, the question is: 
  A) what sequence of read ops at different Vth values maximizes the probability of success before falling back to the expensive algorithm, and 
  B) if read ops fails and our predefined sequence allows for next read try, in which direction we tune Vth and how big of a step we take. 

# Finding optimal sequence / number of read attempts

To answer `A)`, we can begin by asking a question: is there a statistically-naive way of finding a **static** number of tries we can attempt to get the best performance, and while we can come up with a formula, part of that formula will depend on read error rate for previous successful data fetching operations, and that error rate isn't constant (because the drive might age, or environmental factors may change(temp)), therefore the **static** value doesn't exist, and we need to come up with a formula for finding optimal number of attempts based on collected information so far.



# 

- the expensive algorithm can run on a separate thread when the device is idle (assuming it doesnt degrade the disk), and cache the correct Vth for the block
