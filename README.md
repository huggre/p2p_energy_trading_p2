# Integrating physical devices with IOTA — Peer-to-peer energy trading with IOTA Part 2

## The 19th part in a series of beginner tutorials on integrating physical devices with the IOTA protocol.

![Image for post](https://miro.medium.com/max/1700/1*sd4aNIBeQ9jEVbLtOOGbrA.png)

## Introduction

This is the 19th part in a series of beginner tutorials where we explore integrating physical devices with the IOTA protocol. This the second part in a two part tutorial where we look at using the IOTA token as the settlement currency in a peer-to-peer energy trading solution. In the [first part](https://medium.com/coinmonks/integrating-physical-devices-with-iota-peer-to-peer-energy-trading-with-iota-part-1-b86a1e173328) we focused on monitoring real time power consumption and making payments accordingly. In this second part we will look at dealing with variable energy prices, a common problem that must be addressed when dealing with renewable energy sources such as solar, wind, hydro etc.

*Note!
If you haven’t already, you should check out the* [*first part of this tutorial*](https://medium.com/coinmonks/integrating-physical-devices-with-iota-peer-to-peer-energy-trading-with-iota-part-1-b86a1e173328) *before continuing as it forms the basis for the project we are building in this tutorial.*

------

## The Use Case

After having there new IOTA based peer-to-peer energy trading solution up and running for some time, our two neighbors again meet to discuss an unexpected problem with there new system. The neighbor explains that due to bad weather the last couple of months his solar power system have not been producing the amount of energy he was hoping for, and that the original energy price (0.2 IOTA per mW/seconds) they both agreed upon is starting to look as a bad deal for him. They could of course simply adjust the price so that the neighbor gets a higher price for his electricity; but what if the weather improves and his solar panels start generating more electricity? In that case the hotel owner ends up with the short end of the stick. What they need is some type of mechanism that dynamically adjusts the price based on the amount of energy the solar panels are producing at any given time.

Let’s see what we can do.

------

## The solution

After thinking about this problem for a few days the hotel owner pitches the following idea:

What if we take a second power sensor and puts it in the circuit between the neighbors solar panels and batteries? We then measure the amount of energy being produced by the solar panels and automatically adjust the price accordingly.

What a great idea, let’s start building..

------

## The components

In this section I will only discuss added or changed components since the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-peer-to-peer-energy-trading-with-iota-part-1-b86a1e173328), please see the [previous tutorial](https://medium.com/coinmonks/integrating-physical-devices-with-iota-peer-to-peer-energy-trading-with-iota-part-1-b86a1e173328) for details about components not listed here.

**INA3221 current/voltage/power sensor**
The INA3221 is 3 channel low cost current/voltage/power sensor that comes in the form of a breakout board that easily connects to the Raspberry PI’s GPIO pins. The great thing about the INA3221 (compared to the INA219 sensor we used in the previous tutorial) is that it has multiple channels that allows us to monitor multiple power sources in parallel. You should be able to get the INA3221 sensor off ebay or amazon for less that 10 USD.

*Note!*
*Another option to using the INA3221 sensor for this project is to use two INA219 sensors.*

![Image for post](https://miro.medium.com/max/600/0*nXAUMN4CO23rdsPT.JPG)

**Screw terminal blocks**
My INA3221 did not come with any screw terminals for the input/output power channels, so i suggest you get some. They simply makes life easier when hooking up the power sources we want to monitor.

![Image for post](https://miro.medium.com/max/600/0*5Y2HlG2IL85gE5xK.jpg)

**Solar panel**
Solar panels comes in all shapes and sizes. For our project we only need a small panel capable of charging our 3V rechargeable battery. When doing this project I did not have any solar panels laying around so I ended up hacking some of my girlfriends solar powered garden lights. You can easily get the amount of power you need to charge the battery by wiring two or more of these small panels in series.

![Image for post](https://miro.medium.com/max/500/0*OqDor51QmQKw97e9.jpg)

**Diode**
Finally we need a small diode placed in the circuit between the solar panel and the battery. Using the diode we make sure that current only flows from the solar panel to the battery, and not vise versa.

*Note!*
*Make sure the diode is wired the correct way in your circuit, otherwise it will not work.*

![Image for post](https://miro.medium.com/max/500/0*ZMQN4-tVotDyi4Yh)

------

## The wiring diagram

The circuit diagram for the second part of the tutorial is basically the same as for the first tutorial, except that we have replaced the single channel INA219 sensor with the multi channel INA3221 sensor. Also, notice how the INA3221 is now placed in the circuit between the solar panel and the battery. This allows us to monitor both the current going from the battery to the LED, and at the same time, the current going from the solar panel to the battery.

![Image for post](https://miro.medium.com/max/1700/1*sd4aNIBeQ9jEVbLtOOGbrA.png)

------

## Required Software and libraries

The following Python libraries are required for this project.

The [PyOTA library](https://github.com/iotaledger/iota.py) for communicating with the IOTA tangle.

The [INA3221 Python library](https://github.com/switchdoclabs/SDL_Pi_INA3221) for communicating with the INA3221 sensor.

*Note!While the INA219 and INA3221 are similar in functionality, they use different on-board electronics that makes the libraries incompatible across the two units.*

------

## The Python Code

The main changes in our Python code from the previous tutorial (beside using a different library) is that we now not only monitor current going from the battery to the LED, but also the current going from the solar panel to the battery. The **mW_price** variable is also changed from a static variable to a dynamic variable that changes depending on the amount of energy being produced by the solar panel. The more energy the panel produces, the lower the price, and vice versa.

And here is the updated Python code..

```python
#!/usr/bin/env python

# Import PyOTA library
import iota
from iota import Address

# Import time
import time

# Import the INA3221 library
import SDL_Pi_INA3221

# Define the INA3221 object
ina3221 = SDL_Pi_INA3221.SDL_Pi_INA3221(addr=0x40)

# Define INA3221 channels
LED_CHANNEL = 1 # The LED is connected to CHN1 on the INA3221 
SOLAR_PANEL_CHANNEL   = 2 # The solar panel is connected to CHN2 on the INA3221

# Hotel owner IOTA seed used for payment transaction
# Replace with your own devnet seed
seed = b"YOUR9SEED9GOES9HERE"

# Solar panel owner reciever address
# Replace with your own devnet reciever address
addr = "MICIKTVQFXDBZARARUUBXY9OBFDCOFBTYXGOWBWYFZIPYVZVPDLMBVKRF9EUSFASVECRT9PBVBMWMZWADPWZPDDLOD"

# URL to IOTA fullnode used when interacting with the Tangle
iotaNode = "https://nodes.devnet.iota.org:443"

# Define IOTA api object
api = iota.Iota(iotaNode, seed=seed)

# Function for retrieving current power consumption from the LED
def get_current_LED_mW():
    try:
        led_voltage_mV = ina3221.getShuntVoltage_mV(LED_CHANNEL)
        led_current_mA = ina3221.getCurrent_mA(LED_CHANNEL)
        led_watts_mW = led_current_mA * led_voltage_mV
        print("Current LED Power Consumption: %.3f mW" % led_watts_mW)
        return led_watts_mW
    except DeviceRangeError as e:
        print(e)

# Function for retrieving current power generation from the solar panel
def get_current_PANEL_mW():
    try:
        panel_voltage_mV = ina3221.getShuntVoltage_mV(SOLAR_PANEL_CHANNEL)
        panel_current_mA = ina3221.getCurrent_mA(SOLAR_PANEL_CHANNEL)
        panel_watts_mW = panel_current_mA * panel_voltage_mV
        print("Current solar panel Power generation: %.3f mW" % panel_watts_mW)
        return panel_watts_mW
    except DeviceRangeError as e:
        print(e)


# Function for sending IOTA payment
def pay(payment_value):
    
    # Display preparing payment message
    print('Preparing payment of ' + str(payment_value) + ' IOTA to address: ' + addr + '\n')
               
    # Create transaction object
    tx1 = iota.ProposedTransaction( address = iota.Address(addr), message = None, tag = iota.Tag(iota.TryteString.from_unicode('HOTELIOTA')), value = payment_value)

    # Send transaction to tangle
    print('Sending transaction..., please wait\n')
    SentBundle = api.send_transfer(depth=3,transfers=[tx1], inputs=None, change_address=None, min_weight_magnitude=9)       

    # Display transaction sent confirmation message
    print('Transaction sendt...\n')
    
# Define some variables
pay_frequency = 60
pay_counter = 0
accumulated_LED_mW = 0.0
accumulated_PANEL_mW = 0.0

# Main loop that executes every 1 second
while True:
    
    # Check if payment round has been completed 
    if pay_counter == pay_frequency:
        
        
        # Calculate average power consumption for this round
        average_LED_mW = accumulated_LED_mW / pay_frequency
        print("*** Average LED Power Consumption: %.3f mW" % average_LED_mW)
        
        # Calculate average solar panel power generation for this round
        average_PANEL_mW = accumulated_PANEL_mW / pay_frequency
        print("*** Average Solar Panel Power Generation: %.3f mW" % average_PANEL_mW)
        
        # Calculate energy price (10 IOTA's devided by the average ammont of power (mW) generated by the solar panel)
        # The more power generated by the solar panel the lower the price, and vise versa.
        mW_price = 10 / average_PANEL_mW # 
        
        # Calculate IOTA payment based on current mW price and average power consumption
        # Discard any decimals
        pay_value = int((average_LED_mW * pay_frequency)*mW_price)
               
        # Send IOTA payment
        pay(pay_value)
        
        # Reset and prepare for next payment round
        accumulated_LED_mW = 0.0
        accumulated_PANEL_mW = 0.0
        pay_counter = 0
       

    # Get current LED power consumption
    current_LED_mW = get_current_LED_mW()
    
    # Add current LED power consumption to accumulated LED power consumption 
    accumulated_LED_mW = accumulated_LED_mW + current_LED_mW
    
    # Get current solar panel power generation
    current_PANEL_mW = get_current_PANEL_mW()
    
    # Add current solar panel power generation to accumulated solar panel power generation 
    accumulated_PANEL_mW = accumulated_PANEL_mW + current_PANEL_mW
    
    # Increase pay counter
    pay_counter = pay_counter +1
    
    # Wait for one second before taking a new reading
    time.sleep(1)
```

You can download the code from [here](https://gist.github.com/huggre/70ae3d2d543f4dcbbb376199772ea910)

------

## Running the project

To run the project, you first need to save the Python code from the previous section to your machine.

*Note!You also need to save the* **SDL_Pi_INA3221.py** *file (library) from the* [*INA3221 github repo*](https://github.com/switchdoclabs/SDL_Pi_INA3221)*. to the same folder.*

To execute the code, simply start a new terminal window, navigate to the folder where you saved the file and type:

**python** [**p2p_energy_trade_p2.py**](https://gist.github.com/huggre/23a2db302eb7adb8b7e71fe3dc7e06cf)

You should now see the current LED power usage, together with the amount of power being generated by the solar panel, printed to the terminal every second.

Notice how the LED power usage value changes as you turn the potentionmeter.

Also notice how the power generation value from the solar panel changes as you apply more light to the panel by holding it out in the sun, or shining on it with a powerful flashlight.

Every 60 seconds, the average LED power usage for the last period (60 seconds) together with the average solar power generation (for the same period) is calculated, before both values are printed to the monitor.

Next, the price (**mW_price)** is calculated based on the average amount of energy being produced by the solar panel for the last period (60 seconds).

Finally, we calculate the IOTA payment for this period (60 seconds) before sending a value transaction to the IOTA tangle.

------

## Whats next?

Next time we will be talking to the IOTA tangle using the Amazon Alexa personal voice assistant. How cool is that!!, stay tuned.

------

## Contributions

If you would like to make any contributions to this tutorial you will find a Github repository [here](https://github.com/huggre/p2p_energy_trading_p2)

------

## Donations

If you like this tutorial and want me to continue making others feel free to make a small donation to the IOTA address below.

![Image for post](https://miro.medium.com/max/382/0*buIAMbunAMXLb3JQ.png)

NYZBHOVSMDWWABXSACAJTTWJOQRPVVAWLBSFQVSJSWWBJJLLSQKNZFC9XCRPQSVFQZPBJCJRANNPVMMEZQJRQSVVGZ
