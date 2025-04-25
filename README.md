# Home Assistant - B2500 Solar Battery - Zero Power Control
This repository describes the steps required to control a Marstek B2500 in Home Assistant.
In addition it is explained how the battery is controlled to achieve ~0 Watt consumption at the electricity meter.

## Hardware Setup
### Hardware Architecture
![image](https://github.com/user-attachments/assets/1d6a6b6c-d82f-44b7-89be-9c71baec4f20)

### Marstek B2500 Solar Battery and inverter
> [!CAUTION]
> Be aware of the risks when handling electricity and request professional support for maintenance on mains voltage.
>
> Keep in mind that even the DC wiring of the solar panels can pose a fire hasard if the incorrect cable diameter is used.
> ![image](https://github.com/user-attachments/assets/e5171700-8e27-4bb1-b5e4-682b9282c4ed)

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
> A 5V USB supply in the fuse box is required to supply the IR reading head.

The IR reading head is used to extract the power reading from the smart meter installed by the electricity provider inside your fuse box.
Measuring the power on the smart meter is required to implement the zero power control in home assitant.
For my setup the [WIFI based Tasmota IR reading head from the company Stromleser](https://amzn.eu/d/bLCDqwN) is used.
There are also other versions from different manufacturers avaliable.
It must be placed on the optical interface of the smart meter.
> [!TIP]
> There are some smart meters with dimm IR transmitters in the optical interface which makes it impossible for the IR reading head to receive information.
> You can check this by filming the optical interface with a camera which makes the emitted IR light visible to the human eye.
> In my case (Logarex smart meter) this required me to increase the sensitivity of the IR reading head by soldering a different resistor to the voltage divider.

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

Personal experience also proves this statement.
To eliminate this problem the B2500 solar battery can also be controlled via MQTT.

#### Step 1: Configure MQTT using the power zero app
> [!WARNING]
> Enabling MQTT will void the warranty of the B2500 solar battery.

By default MQTT is disabled and can be enabled through the [cloud service](https://eu.hamedata.com/app/AfterSales/index.html).

After MQTT is enabled a new section will appear in your settings within the power zero app.
Fill out the data with your MQTT broker details from home assistant.

![image](https://github.com/user-attachments/assets/2ed35e3e-0d36-4224-829c-e9a35228e849)
![image](https://github.com/user-attachments/assets/55eefc9b-aeb3-413c-a5c4-02e06fd184b6)

If the setup was correct you should be able to request and receive data of the B2500 via home assistant.

The current data can be requested by sending the payload ````cd=01```` on the publication topic mentioned in the MQTT configuration of the B2500 hame_energy/HMK-2/App/DEVICE-ID/ctrl:

![image](https://github.com/user-attachments/assets/4d59ef68-932e-40cc-bb06-41c9e98841a2)

The B2500 will return the current data on the subscription topic hame_energy/HMK-2/device/DEVICE-ID/ctrl.
All data is sent in one payload. The exact contents of the payload will depend on the version of the solar battery and will look similar to this:
> p1=1,p2=1,w1=32,w2=27,pe=21,vv=216,sv=17,cs=0,cd=0,am=0,o1=1,o2=1,do=90,lv=240,cj=2,kn=918,g1=120,g2=119,b1=0,b2=1,md=0,d1=1,e1=0:0,f1=23:59,h1=240,d2=0,e2=0:0,f2=23:59,h2=0,d3=0,e3=0:0,f3=23:59,h3=0,sg=0,sp=80,st=0,tl=20,th=21,tc=0,tf=0,fc=202310231502,id=5,a0=10,a1=0,a2=31,l0=5,l1=2,c0=255,c1=4

![image](https://github.com/user-attachments/assets/1edb058c-8b34-4fa5-9359-aedfd23daa93)

#### Step 2: Create entities from the received payload
Since the data from the B2500 is received as a single payload/string, it is required to extract the data from this string and parse it to entities.
> [!TIP]
> In the documentation accessable via the MQTT configuration the contends of the payload are described:
> 
> ![image](https://github.com/user-attachments/assets/58e4276b-2a68-458a-a644-5952af2dfcc3)

With the following entry into the *configuration.yaml* file of home assistant, the B2500 payload is parsed into entities which can be used as sensors.

````
mqtt:
  - sensor:
    - name: "B2500 Solar 1 Input Power"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      value_template: "{{ value.split('w1=')[1].split(',')[0] }}"
      unique_id: "b2500_solar_1_input_power"
      expire_after: 90

    - name: "B2500 Solar 2 Input Power"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      value_template: "{{ value.split('w2=')[1].split(',')[0] }}"
      unique_id: "b2500_solar_2_input_power"
      expire_after: 90

    - name: "B2500 Battery Percentage"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "%"
      device_class: battery
      value_template: "{{ value.split('pe=')[1].split(',')[0] }}"
      unique_id: "b2500_battery_percentage"
      expire_after: 90

    - name: "B2500 Device Version Number"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('vv=')[1].split(',')[0] }}"
      unique_id: "b2500_device_version_number"
      expire_after: 90

    - name: "B2500 Charging Settings"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('cs=')[1].split(',')[0] }}"
      unique_id: "b2500_charging_settings"
      expire_after: 90

    - name: "B2500 Discharge Settings"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('cd=')[1].split(',')[0] }}"
      unique_id: "b2500_discharge_settings"
      expire_after: 90

    - name: "B2500 AM"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('am=')[1].split(',')[0] }}"
      unique_id: "b2500_am"
      expire_after: 90

    - name: "B2500 DOD Discharge Depth"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "%"
      value_template: "{{ value.split('do=')[1].split(',')[0] }}"
      unique_id: "b2500_dod_discharge_depth"
      expire_after: 90

    - name: "B2500 Battery Output Threshold"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('lv=')[1].split(',')[0] }}"
      unique_id: "b2500_battery_output_threshold"
      expire_after: 90

    - name: "B2500 Scene"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('cj=')[1].split(',')[0] }}"
      unique_id: "b2500_scene"
      expire_after: 90

    - name: "B2500 Battery Capacity"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "Wh"
      value_template: "{{ value.split('kn=')[1].split(',')[0] }}"
      unique_id: "b2500_battery_capacity"
      expire_after: 90

    - name: "B2500 Output Power 1"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('g1=')[1].split(',')[0] }}"
      unique_id: "b2500_output_power_1"
      expire_after: 90

    - name: "B2500 Output Power 2"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('g2=')[1].split(',')[0] }}"
      unique_id: "b2500_output_power_2"
      expire_after: 90

    - name: "B2500 Discharge Setting Mode"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('md=')[1].split(',')[0] }}"
      unique_id: "b2500_discharge_setting_mode"
      expire_after: 90
      
    - name: "B2500 Time1 Start Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('e1=')[1].split(',')[0] }}"
      unique_id: "b2500_time1_start_time"
      expire_after: 90

    - name: "B2500 Time1 End Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('f1=')[1].split(',')[0] }}"
      unique_id: "b2500_time1_end_time"
      expire_after: 90

    - name: "B2500 Time1 Output Value"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('h1=')[1].split(',')[0] }}"
      unique_id: "b2500_time1_output_value"
      expire_after: 90

    - name: "B2500 Time2 Start Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('e2=')[1].split(',')[0] }}"
      unique_id: "b2500_time2_start_time"
      expire_after: 90

    - name: "B2500 Time2 End Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('f2=')[1].split(',')[0] }}"
      unique_id: "b2500_time2_end_time"
      expire_after: 90

    - name: "B2500 Time2 Output Value"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('h2=')[1].split(',')[0] }}"
      unique_id: "b2500_time2_output_value"
      expire_after: 90

    - name: "B2500 Time3 Start Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('e3=')[1].split(',')[0] }}"
      unique_id: "b2500_time3_start_time"
      expire_after: 90

    - name: "B2500 Time3 End Time"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('f3=')[1].split(',')[0] }}"
      unique_id: "b2500_time3_end_time"
      expire_after: 90

    - name: "B2500 Time3 Output Value"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "W"
      value_template: "{{ value.split('h3=')[1].split(',')[0] }}"
      unique_id: "b2500_time3_output_value"
      expire_after: 90

    - name: "B2500 Minimum Temperature of Battery Cells"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "°C"
      value_template: "{{ value.split('tl=')[1].split(',')[0] }}"
      unique_id: "b2500_min_temp_battery_cells"
      expire_after: 90

    - name: "B2500 Maximum Temperature of Battery Cells"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "°C"
      value_template: "{{ value.split('th=')[1].split(',')[0] }}"
      unique_id: "b2500_max_temp_battery_cells"
      expire_after: 90

    - name: "B2500 Chip FC4 Version Number"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      value_template: "{{ value.split('fc=')[1].split(',')[0] }}"
      unique_id: "b2500_chip_fc4_version"
      expire_after: 90

    - name: "B2500 Host Battery capacity"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement
      value_template: "{{ value.split('a0=')[1].split(',')[0] }}"
      unique_id: "b2500_host_battery_capacity"
      expire_after: 90

    - name: "B2500 Extra Battery 1 capacity"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement
      value_template: "{{ value.split('a1=')[1].split(',')[0] }}"
      unique_id: "b2500_speichererweiterung1_capacity"
      expire_after: 90

    - name: "B2500 Extra Battery 2 capacity"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement
      value_template: "{{ value.split('a2=')[1].split(',')[0] }}"
      unique_id: "b2500_extra_battery_2_capacity"
      expire_after: 90


  - binary_sensor:
    - name: "B2500 Solar Input Status 1"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('p1=')[1].split(',')[0] }}"
      unique_id: "b2500_solar_input_status_1"
      expire_after: 90

    - name: "B2500 Solar Input Status 2"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('p2=')[1].split(',')[0] }}"
      unique_id: "b2500_solar_input_status_2"
      expire_after: 90

    - name: "B2500 Is Power Pack 1 Connected"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('b1=')[1].split(',')[0] }}"
      unique_id: "b2500_power_pack_1_connected"
      expire_after: 90

    - name: "B2500 Is Power Pack 2 Connected"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('b2=')[1].split(',')[0] }}"
      unique_id: "b2500_power_pack_2_connected"
      expire_after: 90

    - name: "B2500 Output State 1"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('o1=')[1].split(',')[0] }}"
      unique_id: "b2500_output_state_1"
      expire_after: 90

    - name: "B2500 Output State 2"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('o2=')[1].split(',')[0] }}"
      unique_id: "b2500_output_state_2"
      expire_after: 90

    - name: "B2500 Time1 Enable Status"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('d1=')[1].split(',')[0] }}"
      unique_id: "b2500_time1_enable_status"
      expire_after: 90

    - name: "B2500 Time2 Enable Status"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('d2=')[1].split(',')[0] }}"
      unique_id: "b2500_time2_enable_status"
      expire_after: 90

    - name: "B2500 Time3 Enable Status"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('d3=')[1].split(',')[0] }}"
      unique_id: "b2500_time3_enable_status"
      expire_after: 90

    - name: "B2500 Charging Temperature Alarm"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('tc=')[1].split(',')[0] }}"
      unique_id: "b2500_charging_temp_alarm"
      expire_after: 90

    - name: "B2500 Discharge Temperature Alarm"
      state_topic: "hame_energy/HMK-2/device/DEVICE-ID/ctrl"
      payload_on: "1"
      payload_off: "0"
      value_template: "{{ value.split('tf=')[1].split(',')[0] }}"
      unique_id: "b2500_discharge_temp_alarm"
      expire_after: 90
````
> [!TIP]
> The expire time of the entities will help to detect if the connection between home assistant and the B2500 is lost or unstable.

## Control the B2500 via Home Assistant
### Software Architecture


### Determine and transmit the output power setpoint
#### Step 1: Calculate the required output power of the B2500 solar battery
> [!TIP]
> For every of the following helper entities the sensor template is used.

In order to achieve zero power control it is neccessary to determine the power draw of the consumers in the building.
Since the power is measured on the energy meter (````tasmota_lk13be_current````) and also feedback of the output power of the B2500 is received, we can calculate the consumer power draw (````electrical_consumption````) by simple substraction. Keep in mind that the output power of the B2500 is split over two outputs (````b2500_output_power_1```` and ````b2500_output_power_2````).

The following template is used to create the entity ````electrical_consumption````:
````
{{ states('sensor.b2500_output_power_1') | float(0)
+ states('sensor.b2500_output_power_2') | float(0)
+ states('sensor.tasmota_lk13be_current') | float(0) }}
````


The next helper entity calculates the power setpoint ````solar_battery_output_power_setpoint```` based on the intended power on the electricity meter. For the intended power on the electricity meter an input number helper entity is used which enables also setpoints apart from zero.
````
{{ states('sensor.electrical_consumption') | float(0)
- states('input_number.electricity_meter_power_setpoint') | float(0) }}
````
> [!TIP]
> It can be useful to stay slightly above 0 watts as the intended power on the electricity meter.
> This way small fluctuations will not cause energy from the battery being "lost" into the powergrid.
>
> ![image](https://github.com/user-attachments/assets/32e90ef1-616a-4e86-b8d9-a15567674df8)
>
>  This might be relevant if you don't receive any money from the energy provider when feeding energy back into the power grid.
> In this case it is more feasable to keep the energy in your battery until you might need it at night or on rainy days.


When controlling the B2500 solar battery it can be observed, that the requested power setpoint ````b2500_battery_output_threshold```` and the actual output power ````b2500_output_power_total```` of the B2500 do not always match.
This problem is especially noticable for low output power setpoints:

![image](https://github.com/user-attachments/assets/a0150331-3922-46fc-a085-cda20393f12d)

To eliminate this issue, the error between the intended setpoint and the actual output is calculated as a new helper entity ````remaining_error_output_power_setpoint````.
This new entity is later used to compensate the error.
````
{{ [states('sensor.b2500_battery_output_threshold') | float(0)
- states('sensor.b2500_output_power_total') | float(0), 0] | max
if states('binary_sensor.b2500_pv_output_active') == 'on' else 0}}
````


The last entity required is the corrected power setpoint ````solar_battery_output_power_setpoint_corrected````, which is also the setpoint that will be sent via MQTT to the B2500 solar battery. Since the battery is not able to output power below 80 watts or above 800 watts, the setpoint is limited accordingly.
````
{{ [[states('sensor.solar_battery_output_power_setpoint') | float(0)
+ states('sensor.remaining_error_output_power_setpoint') | float(0), 80] | max, 800] | min }}
````

#### Step 2: Send the new output power setpoint to the B2500 Solar Battery periodically
To send new output power setpoints to the B2500 Solar Battery an automation can be used.
The following automation is triggered every time the state of the IR reading head is changed, e.g. a new value is received.
Afterwards new data is requested from the B2500 solar battery by sending the payload ````cd=01````.
Once new data is received and a new setpoint is calculated, a payload is sent to the solar battery with the updated setpoint.
> [!TIP]
> Implementing the automation in this way features the benefit, that it is always triggered once a new power reading from the IR reading head is avaliable.

> [!NOTE]
> The payload sent to set the output power is typically intended to set a constant output power for a given time interval.
> Eventhough the power setpoint is not intended for frequent changes the response time of the B2500 solar battery can keep up with periodic changes every 10 seconds.

````
alias: B2500 Set Output Power
description: ""
triggers:
  - trigger: state
    entity_id:
      - sensor.tasmota_lk13be_current
    enabled: true
conditions: []
actions:
  - sequence:
      - metadata: {}
        data:
          qos: 0
          retain: true
          topic: hame_energy/HMK-2/App/DEVICE-ID/ctrl
          payload: cd=01
        action: mqtt.publish
      - wait_for_trigger:
          - trigger: state
            entity_id:
              - sensor.solar_battery_output_power_setpoint_corrected
        timeout:
          hours: 0
          minutes: 0
          seconds: 30
      - action: mqtt.publish
        metadata: {}
        data:
          retain: true
          topic: hame_energy/HMK-2/App/DEVICE-ID/ctrl
          payload: >-
            cd=07,md=0,a1={{ states('input_number.b2500_enable_timer_1') |
            round(0) }},b1=0:00,e1=23:59,v1={{
            states('sensor.solar_battery_output_power_setpoint_corrected') | round(0)
            }},a2=0,b2=0:00,e2=23:59,v2=0,a3=0,b3=0:00,e3=23:59,v3=0
          qos: 0
mode: single
````
> [!NOTE]
> As you can see there is an additional entity used in the last payload called ````b2500_enable_timer_1````.
> The usage of this entity will explained later on when [automated resets and cell balancing](#enable-and-disable-the-b2500-power-output) are implemented.

#### Step 3: Observe the control strategy
The control via home assistant keeps the measured power on the smart meter at the intended setpoint by compensating the electrical consumption with the output of the B2500 solar battery:

![image](https://github.com/user-attachments/assets/2fee9322-3b1c-4e25-a099-1d7baaf1c910)

> [!TIP]
> The discrete step time is dependend on the update rate of the IR reading head. In my setup it is set to 10 seconds.
> In case you want a different step size you can configure this in the tasmota interface of the IR reading head.

### Enable and disable the B2500 power output
Experience shows that there are situations in which the B2500 solar battery freezes up and requires a reset. In the following example two possible scenarios can be observed.
- At time 15:40: Only one of the two outputs from the B2500 solar battery is activated
- At time 16:02: Both outputs are active but they don't react to the demanded setpoint

![image](https://github.com/user-attachments/assets/38401170-0fcf-4545-a9d3-b66cb8ce8437)

To understand how the reset needs to be performed it is important to understand how the B2500 usually behaves under normal operation and after a reset.
Under normal operating conditions both outputs have approximately the same output power. Which means the commanded output power setpoint is split equally over both outputs.

In case of a reset, the B2500 will behave as follows:
- At time 12:05: The output is disabled and the output power drops to 0 watts. As soon as the output power reaches 0 watts, the output is activated again
- At time 12:12: The B2500 activates both outputs at a low power setting (````b2500_output_power_total```` is below 80 watts)
- At time 12:19: The total output power increases to the output power setpoint

![image](https://github.com/user-attachments/assets/1c50d07f-a94f-41fd-bc5a-529a3f9f4e4f)

> [!IMPORTANT]
> As shown the startup/reset process takes some time.
> For this reason home assistant shouldn't reset the B2500 while it is still occupied by the startup process.
> This would lead to a never-ending reset loop.

#### Disable the B2500 output
The following automation is used to disable output of the B2500. It can be split into two functions.
##### 1. Reset due to faulty output
In case the output power of the B2500 is above 10 watts but significantly below the setpoint in home assistant for more than 10 minutes, the output will be deactivated.

Similarly, the output will be deactivated if the output power of the B2500 is above 10 watts but significantly below the setpoint stored within the B2500 itself for more than 12,5 minutes.

##### 2. Disable to cell balancing
Another reason for the deactivation of the output is the cell balancing functionality. If cell balancing is active and there is no excess solar power, the output will be disabled. 
````
alias: B2500 Disable Output
description: ""
triggers:
  - trigger: numeric_state
    entity_id:
      - sensor.b2500_output_power_total
    above: 30
    below: sensor.solar_battery_output_power_setpoint
    value_template: "{{ float(state.state) + 20}}"
    for:
      hours: 0
      minutes: 10
      seconds: 0
    id: Deviation from HA-Setpoint
  - trigger: numeric_state
    entity_id:
      - sensor.b2500_output_power_total
    above: 30
    below: sensor.b2500_time1_output_value
    value_template: "{{ float(state.state) + 20}}"
    for:
      hours: 0
      minutes: 12
      seconds: 30
    id: Deviation from B2500-Setpoint
  - trigger: state
    entity_id:
      - timer.b2500_cell_balancing
    to: idle
    id: Cell balancing
  - trigger: numeric_state
    entity_id:
      - sensor.b2500_solar_total_input_power
    for:
      hours: 0
      minutes: 15
      seconds: 0
    id: No excess solar power
    below: sensor.electrical_consumption
conditions:
  - condition: or
    conditions:
      - condition: trigger
        id:
          - Deviation from HA-Setpoint
          - Deviation from B2500-Setpoint
          - Cell balancing
      - condition: and
        conditions:
          - condition: trigger
            id:
              - No excess solar power
          - condition: state
            entity_id: timer.b2500_cell_balancing
            state: idle
actions:
  - action: input_number.set_value
    metadata: {}
    data:
      value: 0
    target:
      entity_id: input_number.b2500_enable_timer_1
mode: single
````

#### Enable the B2500 output
````
alias: B2500 Set Output
description: ""
triggers:
  - trigger: numeric_state
    entity_id:
      - input_number.b2500_enable_timer_1
    below: 1
  - trigger: state
    entity_id:
      - timer.b2500_cell_balancing
    to: active
  - trigger: numeric_state
    entity_id:
      - sensor.b2500_solar_total_input_power
    for:
      hours: 0
      minutes: 10
      seconds: 0
    value_template: "{{ float(state.state) - 100}}"
    above: sensor.electrical_consumption
    id: Excess solar power
conditions:
  - condition: or
    conditions:
      - condition: state
        entity_id: timer.b2500_cell_balancing
        state: active
      - condition: trigger
        id:
          - Excess solar power
actions:
  - wait_for_trigger:
      - trigger: numeric_state
        entity_id:
          - sensor.b2500_output_power_total
        below: 10
    timeout:
      hours: 0
      minutes: 1
      seconds: 0
  - action: input_number.set_value
    metadata: {}
    data:
      value: 1
    target:
      entity_id: input_number.b2500_enable_timer_1
mode: single
````
