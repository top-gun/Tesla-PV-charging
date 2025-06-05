# Tesla-PV-charging

## Description:

Home-Assistant can talk to all common photovoltaic systems, and it can talk to your Tesla car. So let's use Home-Assistant to charge the car with your spare PV power! :) 

This documents my solution to charge my Tesla with PV excess. I own a Tesla Model 3, a Huawei PV system with 8.5kW peak and 10kWh battery. The Tesla is plugged into a "dumb" wallbox that can charge with 11kW (3-phase). All charge control is done via the car API, the wallbox is dumb and does just safety stuff. You plug the car into the wallbox, and it turns on the power.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/GUI-advanced.png">

My aim is to both use the home-battery as a buffer to allow smoother charging of my Tesla, and to make sure that my house battery is charged to 90-100% at the end of the day. Also, it allows me to use a simple wall-box which is around 350 Euro vs the not-so-smart boxes in the 600-1000 Euro-range. We can control the charging power in the range of 0.7kW to 11kW, that's smoother than the expensive wall boxes and we don't need to fuss around with 1/3-phase-charging and the issues many boxes have when they switch mode.

This approach works well for me - I have run my Tesla for 33,000km so far, and except during the winter, it operates solely on solar excess power. 

Why did I go the extra mile, when there are ready-made solutions? I looked into EVCC - it relies on smart (=expensive and complicated) wall-boxes, and is limited by the capabilities of the wallbox. It can't charge slower than the wallbox allows, which is an issue especially if the box can't switch 1/3-phase-charging. I also looked into Tesla Solar Charger. It doesn't need a special wallbox, but with Tesla charging for their API, Patrick had to charge for the product, too.

## How does it do that?

It monitors the PV output from the roof and the current state of the house battery. There are three main stages in the daily charging cycle:

- When the house battery is below 40%, charging the house battery has full priority. No charging of the car is done, except when the PV output is very high and the house battery can't take all the power.
- When the house battery is between 40% and 75%, 70% of the unused PV power is used to charge the car, the rest is taken by the house battery. Since the house battery is just a fraction of the car battery, that quota is generally enough to fill the house battery over the day.
- When the house battery is between 75% and 90%, the car is charged at 80% solar output but keeping a minimum of 4kW (6Ax3) to speed up charging.
- When the house battery is charged beyond 90%, the car gets the full surplus. Since we round off to the next lower ampere setting, some power will still go into the house battery.
- Hysteresis: For the transition between the three stages, a hysteresis of 3-5% is built into each formula to make sure don't continually jump up and down with every cloud on the sky. This is where the house battery comes very handy: We can afford to smooth the current a bit by using the household battery as a buffer.
- Charge with 0A: This is something the "smart wallboxes" can't do. When the PV output goes down due to clouds, we tell the car to charge at 0A, but keep the connection alive. We don't want the car to open and close the big relais every few minutes when the weather is shady.
- Express mode: I added an "Express mode" which aims to charge the car battery as fast as possible and uses house battery energy in addition to solar output. This is practical for days when I need to head to the office, but expect lots of sunshine. It makes sense to shift battery power into the car and then have the house battery recharge during the day. This stops automatically at 20% SOC of the house battery. The formula does also minds the combined maximum inverter power. My PV can deliver 8.5kW max, my house battery can deliver 5kW, but the inverter is limited to 10kW maximum power. So the consumption has to stay below 10kW to avoid grid import.

## Prerequisites:

You should own a photovoltaic system with enough power to charge your car, with a reasonably sized battery. My system has 8.5kW peak power and a 15kWh battery. You can go lower on the PV power, but it will take longer to charge your car, obviously. The size of the battery is not really important. It will just work as a buffer and even a small 3kWh unit should be just fine. My system is made by Huawei, a Sun2000-10-KTL-M1 with their matching smart-meter and their matching Luna2000 battery. Since I don't use any proprietary functions, it is easy to adapt. I use standard numbers like inverter input power, battery SOC and so on that any standard PV system will provide.

A wallbox (if you charge 3-phase) or the Tesla AC charger that plugs into any ac wallplug. There is no need for the wallbox to have any management capabilities. In fact, this stuff works easier with a wallbox that just charges and doesn't require any commands to do its job.

Then you want Home-Assistant as a house automation system. It is the foundation for what we do here. Home-Assistant has plenty of integrations into the typical house stuff like wall plugs, heating valves and so on. There are integrations for all the common PV brands through a community-managed integration appstore called HACS. You can run Home-Assistant on a Raspberry Pi or similar small computers. The company behind Home-Assistant does also sell ready-to-use systems with their own hardware. If you can find or have a Raspberry Pi at hand, it's cheaper and just fine.

ESP32 for BLE-control: An inexpensive (5-8 Euro) device to avoid the cloud charges from Tesla. Since 2025, Tesla charges for the use of their cloud API. Commands are especially expensive, and since PV output varies wildly with every cloud on the sky, we send a lot of commands. Therefore, I switched to Bluetooth (BLE)-control of the car. What you need is a cheap mini-computer "ESP32" which is available on Amazon or Ali Express for about 5 Euro/USD. The simplest ones are called "NodeMCU Devkit-C" and are adequate for the job. I recommend to buy from Amazon as it's faster, but you can save a bit ordering from AliExpress. I recommend these: https://www.amazon.de/diymore-NodeMCU-Nodemcu-Development-Bluetooth/dp/B0C6QHLGJG . If you find one without the soldered pins it's actually more practical because you only plug in a USB-C power cable. These have a modified processor. Some are labelled as ESP32-C3. You can use them, it needs just a few extra lines in the configuration later on in this document. Each ESP32 gets paired to one vehicle. If you own two Teslas, get two ESP32-dongles. They are usually sold in packs of two or three anyway.

## Steps:

### 1. Installation: I assume you have a working Home Assistant installation
   
   1.1 You need to install HACS (Home Assistant Community Store), it's needed to install most PV integrations, and for some fancy stuff we will use for the Dashboard.

   1.2 You should install "Studio Code Server", it's a comfortable editor for the Home Assistant configuration files. 

   1.3 Tesla control: In Home Assistant, you need the Add-On "Home-Assistant ESP Device Builder". This programs the ESP32 dongle that will talk to your car, controlling the charge process. I wrote an extra guide that explains how to set up the Add-on, program your ESP32 for the task, and how to pair the ESP32 to your car. The guide is here: https://docs.google.com/document/d/1W33jPQH4uAfSzb8rHvg7sBFm9zY_D-149qrtlnRN2xQ/edit?usp=sharing 
Follow the instructions to the end. You will now be able to see your Tesla in Home Assistant as a new device, see the SOC and if it's charging, and you can manually control the charger, even setting the charing current in Amps. We will learn how to automatically charge the car later on :)
   
   1.4 In HACS, install the integration for your PV system and connect it with your local PV system. You don't need all the bells an whistles, the numbers for PV output in Watt and the state of charge of your house battery in percent is all you really need for this. If you can configure the update intervall, 30s is a good starting point.

### 2. Get familiar with the integrations: 

Get to know the Tesla and PV integrations before you continue. Especially, you should get a feeling on how fast the integrations update and check if your Tesla is recognised when you get home. The BLE-radio on the ESP32 works pretty well. I can use it from inside my living room to control the car, but every house is different. If your house is made with lots of steel-reinforced concrete, you may need to move the ESP32 closer to the car. It needs only a 5V handy-charger and wifi coverage to talk with your Home Assistant-server.

### 3. Sensors: Now, we will create a few sensors and variables that are needed to control the PV current.
   3.1 In Settings/Devices/Helpers, create the following helpers. Keep in mind: My car is called "Tesla BLE F549C4" in HA. You may call your car "Tin Lizzy" or "Thors Hammover", that's fine, but the names of your entities will follow your creative outbreak. 
   
Chose the type "toggle", set to off as a starting point:

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Helper-Hand-Control.png" width=300>

Chose the type "number", set to 0 as a starting point:

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla_Charge_Break.png" width=300>
   
Chose the type "toggle", set to off as a stating point:

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Helper-Express-Charge.png" width=300>

Chose the type "Threshold sensor":

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Threshold_sensor_no_charge.png" width=300>

   

   3.2 In Studio Code Server, open the file "configuration.yaml" and add the following sensors. 

   **Attention:**

- The entities sensor.battery_state_of_capacity and sensor.inverter_input_power are specific to my Huawei PV system. If you run a Fronius, SolarEdge, Victron, Sungrow or whatever PV system, you will need to find the right entity names for your system.
- My car is called "Tesla BLE F549C4", therefore the entities for my car have "tesla" after the dot. If your cars name is "godzilla", you need to change that to ie sensor.godzilla_charger_power . 
```
template:
  - sensor:
    - name: 'Autocharge-optimal'
      unit_of_measurement: "A"
      state_class: measurement
      state: > 
        {# calculate the optimal charge current based on several parameters: #}
        {# PV yield, house battery SOC, vehicle battery SOC, Grid consumption #}
        {# PV yield is the current output of the pv system #}
        {% set PV = states('sensor.inverter_input_power')|float * 1000 -500 %}
        {# HP is the power requirement of my house heatpump. I set this to 0 in the reference config. #}
        {% set HP = 0|float %}
        {# set HP = states('sensor.wp_strom_total_active_power')|float #}
        {# PV Battery is my house battery in the PV system #}
        {% set Battery = states('sensor.battery_state_of_capacity')|float (0) %}
        {# PV BatteryMaxDischarge is a limit to the house battery in Express Mode #}
        {% set BatteryMaxDischarge = 4500 %}
        {# Grid is positive when we export and negative if we import #}
        {% set Grid = states('sensor.power_meter_active_power') |float (0) %}
        {# current charging power of the Tesla #}
        {% set Charge = states('sensor.tesla_ble_f549c4_charge_power')|float %}
        {# current SOC of the vehicle #}
        {% set Teslabattery = states('sensor.tesla_ble_f549c4_charge_level') |float (0) %}

        {% if Grid>0 %} {% set Grid = 0 %} {% endif %}
        {# PV/230 is the current in a one phase system. For three-phase charging divide by 3! #}
        {% set PVAMP = ((PV-HP)/230/3) %}
        {% if Battery>97 %} {% set PVAMP = PVAMP+1 %} {% endif %}
        {# While the house battery is below 90% ist, use only 70% of the output so the house battery will charge, too #}
        {% if Battery<90 and Battery>65 %} {% set PVAMP = PVAMP * 0.75 %} {% endif %}
        {% if Battery<=65 %} {% set PVAMP = PVAMP*0.65 %} {% endif %}
        {% if (Teslabattery>90) and (Battery<85) %} {% set PVAMP = PVAMP * 0.8 %} {% endif %}
        {# Under 30%, charge only the house battery, not the car. 3% hysteresis: Don't turn on until we reach 43%. #}
        {# Exception: When the PV output is really high, allow charging so we don't feed to the grid early #}
        {% if (Battery<30) and (Charge==0) and (PVAMP<6)  %} {% set PVAMP = 0 %} {% endif %} 
        {# Don't start charging under 3A because it's not efficient. #}
        {% if (PVAMP<3) and (Charge==0) %} {% set PVAMP = 0 %} {% endif %}
        {# If we pull from the grid, adjust the charge current accordingly. In my system, Gridimport is a negative number #}
        {% if Grid < -300 %} {% set PVAMP = PVAMP + (Grid/230/3) %} {% endif %}
        {% if is_state('input_boolean.tesla_express', 'on') and (Battery>20) %} {% set PVAMP = ((PV + BatteryMaxDischarge + Grid)/230/3) |int %} {% endif %}
        {# My pv system can only deliver 10kW  so we cut off at 14A #}
        {% if PVAMP>14 %} {% set PVAMP = 14%} {% endif %}
        {# avoid negative numbers. #}
        {% if PVAMP<0 %} {% set PVAMP = 0%} {% endif %}
        {# tesla_gridladen is a command to load the car from the grid, I use that for Octopus #}
        {# if is_state('input_boolean.tesla_gridladen', 'on') %} {% set PVAMP = 16 %} {% endif #}
        {{ PVAMP|int }}

  - sensor:
    - name: 'Autocharge-Difference'
      unit_of_measurement: "A"
      state: > 
        {# if there is a difference between optimal charge current and actual charge current, calculate the absolute value #}
        {% set CHARGE = states('number.tesla_ble_f549c4_charging_amps')|float %}
        {% set REQUIRED = states('sensor.autocharge_optimal') |float (0) %}
        {% set DIFF = (CHARGE - REQUIRED)|abs %}
        {{ DIFF|int }}

```

### 4. Automations:

We need several automations. Unfortunately, they can grow pretty long, and there is no really good way to export them. So sorry if I can only give you some very huge screenshots.

   4.1 Tesla-Charge-Adjust: This most important one will simply start every 60s and, after checking the car is at home and wired for charging, set the right current and start or stop the charging process.

These automations refer to Tesla-BLE-F549C4, you need to adjust the name.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla-Charge-Adjust.png" width=600>



   4.2 Tesla-Leaving: When the car leaves home, set the charge mode to automatic and the house battery to 75% minimum for fast charging.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/IMG_1275.png" width=300>

   4.3 Tesla-1800-charge-socket-90p: No longer needed and removed,

   4.4 Tesla-throttle-high-load: No longer needed and removed. This is now handled in the calculation of the optimum current.


### 5. Fancy visualization:

Basically, your car can charge without any intervention. Still, we like to monitor and control, so I made a dashboard for this. It requires two extra front-end extensions which you can download in HACS:

   5.1 Power Flow Card Plus: It shows you the flow of energy in your house, including the charging power to the car.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/GUI-Power-flow-card.png" width=300>

   My definition. As always, you may need to adjust sensor entities if your PV is from a different brand:

```
type: custom:power-flow-card-plus
entities:
  battery:
    entity: sensor.battery_charge_discharge_power
    state_of_charge: sensor.battery_state_of_capacity
    invert_state: true
  grid:
    entity: sensor.power_meter_active_power
    invert_state: true
  solar:
    entity: sensor.inverter_input_power
    display_zero_state: true
  individual:
    - entity: sensor.tesla_ble_f549c4_charge_power
      display_zero: true
      name: Tesla
      icon: mdi:car
      secondary_info:
        entity: sensor.tesla_ble_f549c4_charge_level
        unit_of_measurement: "%"
      unit_white_space: true
      use_metadata: false
  fossil_fuel_percentage:
    secondary_info: {}
  home:
    secondary_info: {}
clickable_entities: true
display_zero_lines: true
use_new_flow_rate_model: true
w_decimals: 0
kw_decimals: 1
min_flow_rate: 0.75
max_flow_rate: 6
max_expected_power: 2000
min_expected_power: 0.01
watt_threshold: 1000
transparency_zero_lines: 0
```

   5.2 Mini-Graph-Card: It shows a graph with the car's SOC and the charging power over the last six hours (adjustable).

<img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Mini-Graph-Card.png" width=300>

My configuration:

```
type: custom:mini-graph-card
entities:
  - entity: sensor.tesla_battery
  - entity: sensor.tesla_ble_f549c4_charge_power
    name: Tesla Charge Power
    y_axis: secondary
    show_state: true
hours_to_show: 6
points_per_hour: 30
upper_bound: 100
lower_bound: 0
upper_bound_secondary: 11
smoothing: false
extrema: true
```
   5.3 Current Tesla Status: This is a regular card of the type "entities".



   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Charge-Status.png" width=300>


The definition:

```
type: entities
entities:
  - entity: binary_sensor.tesla_ble_f549c4_charge_flap
    secondary_info: last-changed
    name: Onboardlader
  - entity: sensor.autocharge_optimal
    name: Empfohlener Ladestrom
  - entity: sensor.tesla_actual_amps
    name: Ladestrom jetzt
    secondary_info: last-changed
  - entity: counter.charge_current_set
    name: Änderungen heute
  - entity: binary_sensor.tesla_ble_f549c4_asleep
    name: Tesla schläft
    secondary_info: last-changed
  - entity: sensor.tesla_ble_f549c4_ble_signal
    name: BLE Signal
  - entity: sensor.tesla_energy_added
  - entity: sensor.tesla_ble_f549c4_charge_level
  - entity: sensor.tesla_ble_f549c4_range
state_color: true
show_header_toggle: false
title: Tesla Status
layout_options:
  grid_columns: 4
  grid_rows: auto
```

   5.4 Tesla control: Information about the charging process, also switches that change to manual charge control (app) or express charging.

   These are actually three horizontal stacks, but at any given time, only two are visible.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Visual-horizontal-stack.png" width=300>

   First row: (The permanent one)
```
type: horizontal-stack
cards:
  - type: custom:button-card
    color_type: card
    show_state: "yes"
    name: Manuell
    entity: input_boolean.auto_manuell
  - type: custom:button-card
    color_type: card
    show_state: "yes"
    name: Express
    entity: input_boolean.tesla_express
  - type: custom:button-card
    entity: number.tesla_ble_f549c4_charging_limit
    show_state: true
    name: Ladelimit
  - type: custom:button-card
    color_type: card
    color: rgb(250,250,250)
    name: Klappe
    state_display:
      - - - öffnen
    show_state: true
    icon: mdi:lock-open
    tap_action:
      action: call-service
      service: automation.trigger
      target:
        entity_id: automation.tesla-ladeklappe
      data:
        skip_condition: true
  - type: custom:button-card
    name: Update
    icon: mdi:reload
    entity: button.tesla_ble_f549c4_force_data_update
    show_state: true
    state_display: jetzt
    tap_action:
      action: call-service
      service: button.press
      service_data:
        entity_id: button.tesla_ble_f549c4_force_data_update
```
   second row (for manual charging)

```
type: horizontal-stack
cards:
  - type: custom:button-card
    color_type: card
    show_state: true
    show_icon: true
    name: Charger
    size: 15%
    entity: switch.tesla_ble_f549c4_charger
  - type: custom:button-card
    name: Strom
    entity: number.tesla_ble_f549c4_charging_amps
    show_state: true
    size: 15%
  - type: custom:button-card
    name: Status
    entity: sensor.tesla_ble_f549c4_charging_state
    show_state: true
    size: 15%
visibility:
  - condition: state
    entity: input_boolean.auto_manuell
    state: "on"
```

   third row (automatic operation)

```
type: horizontal-stack
cards:
  - type: custom:button-card
    color_type: card
    show_state: true
    show_icon: true
    name: Auto schläft
    icon: mdi:sleep
    entity: binary_sensor.tesla_ble_f549c4_asleep
    size: 10%
    state:
      - value: "off"
        styles:
          card:
            - background-color: lightgreen
        icon: mdi:sleep-off
      - value: "on"
        styles:
          icon:
            - color: lightyellow
  - type: custom:button-card
    color_type: card
    show_state: true
    show_icon: true
    icon: mdi:battery-charging
    size: 10%
    name: Laden
    state:
      - value: Charging
        styles:
          card:
            - background-color: lightgreen
      - value: Stopped
        styles:
          icon:
            - color: grey
      - value: Completed
        styles:
          icon:
            - color: gold
    entity: sensor.tesla_ble_f549c4_charging_state
    tap_action:
      action: more-info
visibility:
  - condition: state
    entity: input_boolean.auto_manuell
    state_not: "on"

```
