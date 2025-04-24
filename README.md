# Home Assistant - B2500 Solar Battery - Zero Power Control
This repository describes the steps required to control a Marstek B2500 in Home Assistant.
In addition it is explained how the battery is controlled to achieve ~0 Watt consumption at the electricity meter.

## Hardware Setup
### Hardware Architecture
![image](https://github.com/user-attachments/assets/1d6a6b6c-d82f-44b7-89be-9c71baec4f20)

### Marstek B2500 Solar Battery and inverter
In my case the battery and inverter are placed inside the house to keep the operational temperature between 20°C and 30°C independent of the time of the year.
The inverter is placed right next to the battery DC output to reduce DC-cabling length (which features higher current and thus higher losses).
The AC output of the inverter is connected to a wall outlet.

![image](https://github.com/user-attachments/assets/d72f79e1-0507-4dd4-8dd7-9a73d66bdb70)

### PV-Panels
4 standard PV-Panels are used. Since the battery only features two inputs, the panels are devided into two parallel pairs.
As shown in the picture, the DC cables of the panels pass through the wall and connect to the DC input of the B2500 solar battery.

![image](https://github.com/user-attachments/assets/096564f8-7fe3-4817-8827-a38a02c69207)
![image](https://github.com/user-attachments/assets/3f0fcf1c-3ad3-4f27-b644-25834968ec8a)

### IR reading head
> [!IMPORTANT]
> You will need a 5V USB supply in your fuse box for the IR reading head.

The IR reading head is used to extract the power reading from the smart meter installed by the electricity provider inside your fuse box.
Measuring the power on the smart meter is required to implement the zero power control in home assitant.
For my setup the [WIFI based Tasmota IR reading head from the company Stromleser](https://amzn.eu/d/bLCDqwN) is used.
There are also other versions from different manufacturers avaliable.
It must be placed on the optical interface of the smart meter.
> [!NOTE]
> There are some smart meters with dimm IR transmitters in the optical interface which makes it impossible for the IR reading head to receive information.
> You can check this by filming the optical interface with a camera which makes the emitted IR light visible.
> In my case (Logarex smart meter) this required me to adjust the sensitivity of the IR reading head by soldering a different resistor to the voltage divider.

## Setting up the components
###  IR reading head
To set up the Tasmota IR reading head you can simply follow the [stromleser documentation](https://stromleser.de/en/blogs/news/stromleser-wifi-einrichten-und-einen-stromleser-ttl-lese-kopf-hinzufugen-mit-tasmota).
Depending on your smart meter you will need different [SML-Scripts](https://tasmota.github.io/docs/Smart-Meter-Interface/#smart-meter-descriptors) to read the data from your smart meter.
> [!IMPORTANT]
> You might have to enter a PIN before your smart meter shows detailed information that is required for the zero power control (current power).
> This PIN is either in your energy provider documents can be requested from your energy provider.

### B2500 Solar Battery
