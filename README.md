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
> [!TIP]
> There are some smart meters with dimm IR transmitters in the optical interface which makes it impossible for the IR reading head to receive information.
> You can check this by filming the optical interface with a camera which makes the emitted IR light visible.
> In my case (Logarex smart meter) this required me to adjust the sensitivity of the IR reading head by soldering a different resistor to the voltage divider.

## Integrate the components to Home Assistant
> [!IMPORTANT]
> Make sure to install the [MQTT integration](https://www.home-assistant.io/integrations/mqtt/) before you attempt to connect the IR reading head and the B2500 solar battery to your home assistant.

###  IR reading head
To set up the Tasmota IR reading head you can simply follow the [stromleser documentation](https://stromleser.de/en/blogs/news/stromleser-wifi-einrichten-und-einen-stromleser-ttl-lese-kopf-hinzufugen-mit-tasmota).
Depending on your smart meter you will need different [SML-Scripts](https://tasmota.github.io/docs/Smart-Meter-Interface/#smart-meter-descriptors) to read the data from your smart meter.
> [!TIP]
> You might have to enter a PIN before your smart meter shows detailed information that is required for the zero power control (current power).
> This PIN is either in your energy provider documents can be requested from your energy provider.

### B2500 Solar Battery
By default the B2500 Solar Battery is controlled by the "Power Zero" app.
This app allows for timebased control of the battery, e.g. discharging with a certain power for a given time period.
There is also the possibility to connect a marstek smart meter which features current clamps to measure the current consumption of your building.
Unfortunately the response time and accuracy of this implemented solution could be improved.
As you can see in the description in the app there can be deviations in the intended power of 100 Watts.

![image](https://github.com/user-attachments/assets/ce7dc5b4-0ca9-4657-b86d-b01be6329350)

To eliminate this problem the B2500 solar battery can also be controlled via MQTT.
By default MQTT is disabled and can be enabled through the [cloud service](https://eu.hamedata.com/app/AfterSales/index.html).
> [!WARNING]
> Enabling MQTT will void the warranty of the B2500 solar battery.

After MQTT is enabled a new section will appear in your settings within the power zero app.
Fill out the data with your MQTT broker details from home assistant.

![image](https://github.com/user-attachments/assets/2ed35e3e-0d36-4224-829c-e9a35228e849)
![image](https://github.com/user-attachments/assets/55eefc9b-aeb3-413c-a5c4-02e06fd184b6)

If the setup was correct you should be able to request and receive data of the B2500 via home assistant.

Request data:

![image](https://github.com/user-attachments/assets/4d59ef68-932e-40cc-bb06-41c9e98841a2)


Received data:

![image](https://github.com/user-attachments/assets/1edb058c-8b34-4fa5-9359-aedfd23daa93)
