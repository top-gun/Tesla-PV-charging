# Tesla-PV-charging

## Description:

Home-Assistant can talk to all common photovoltaic systems, and it can talk to your Tesla car. So let's use Home-Assistant to charge the car with your spare PV power! :) 

This documents my solution to charge my Tesla with PV excess. I own a Tesla Model 3, a Huawei PV system with 8.5kW peak and 10kWh battery. The Tesla is plugged into a "dumb" wallbox that can charge with 11kW (3-phase). All charge control is done via the car API, the wallbox is dumb and does just safety stuff. You plug the car into the wallbox, and it turns on the power.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/GUI-Tesla.png">

My aim is to both use the home-battery as a buffer to allow smoother charging of my Tesla, and to make sure that my house battery is charged to 90-100% at the end of the day. Also, it allows me to use a simple wall-box which is around 350 Euro vs the not-so-smart boxes in the 600-1000 Euro-range. We can drive the charing power in the range of 0.7kW to 11kW, that's smoother than the expensive wall boxes and we don't need to fuss around with 1/3-phase-charging and the issues many boxes have when they switch mode.

This approach works well for me - I have run my Tesla for 1,500km, and this was all done with energy from my roof. 

Why did I go the extra mile, when there are ready-made solutions? I looked into EVCC - it relies on smart (=expensive and complicated) wall-boxes, and is limited by the capabilities of the wallbox. It can't charge slower than the wallbox allows, which is an issue especially if the box can't switch 1/3-phase-charging. I also looked into Tesla Solar Charger. It doesn't need a special wallbox, but its charging strategies are not very smart when it comes to balancing energy between house- and vehicle-battery.

## How does it do that?

It monitors the PV output from the roof and the current state of the house battery. There are three main stages in the daily charging cycle:

- When the house battery is below 40%, charging the house battery has full priority. No charging of the car is done, except when the PV output is very high and the house battery can't take all the power.
- When the house battery is between 40% and 75%, 80% of the unused PV power is used to charge the car, the rest is taken by the house battery. Since the house battery is just a fraction of the car battery, that quota is generally enough to fill the house battery over the day.
- When the house battery is between 75% and 90%, the car is charged at 80% solar output but keeping a minimum of 4kW (6Ax3) to speed up charging.
- When the house battery is charged beyond 90%, the car gets the full surplus. Since we round off to the next lower ampere setting, some power will still go into the house battery.
- Hysteresis: For the transition between the three stages, a hysteresis of 3-5% is built into each formula to make sure don't continually jump up and down with every cloud on the sky. This is where the house battery comes very handy: We can afford to smooth the current a bit by using the household battery as a buffer.
- Charge with 0A: This is something the "smart wallboxes" can't do. When the PV output goes down due to clouds, we tell the car to charge at 0A, but keep the connection alive. We don't want the car to open and close the big relais every few minutes when the weather is shaky. This reduces wear on the car charger.

## Prerequisites:

You should own a photovoltaic system with enough power to charge your car, with a reasonably sized battery. My system has 8.5kW peak power and a 10kWh battery. You can go lower on the PV power, but it will take longer to charge your car, obviously. The size of the battery is not really important. It will just work as a buffer and even a small 3kWh unit should be just fine. My system is made by Huawei, a Sun2000-10-KTL-M1 with their matching smart-meter and their matching Luna2000 battery. Since I don't use any proprietary functions, it is easy to adapt. I use standard numbers like inverter input power, battery SOC and so on that any standard PV system will provide.

A wallbox (if you charge 3-phase) or the Tesla AC charger that plugs into any ac wallplug. There is no need for the wallbox to have any management capabilities. In fact, this stuff works easier with a wallbox that just charges and doesn't require any commands to do its job.

Then you want Home-Assistant as a house automation system. It is the foundation for what we do here. Home-Assistant has plenty of integrations into the typical house stuff like wall plugs, heating valves and so on. There are integrations for all the common PV brands through a community-managed integration appstore called HACS. You can run Home-Assistant on a Raspberry Pi or similar small computers. The company behind Home-Assistant does also sell ready-to-use systems with their own hardware. If you can find or have a Raspberry Pi at hand, it's cheaper and just fine.

## Steps:

### 1. Installation: I assume you have a working Home Assistant installation
   
   1.1 You need to install HACS (Home Assistant Community Store), it's needed to install the integrations for your PV system and for the Tesla car.

   1.2 You should install "Studio Code Server", it's a comfortable editor the the Home Assistant configuration files. 

   1.3 In HACS, install the Tesla integration and connect it with your Tesla car. There is a good documentation in the HACS page, especially for creating the "tokens" you need to let HA talk with your car.

   1.4 In HACS, install the integration for your PV system and connect it with your local PV system. You don't need all the bells an whistles, the numbers for PV output in Watt and the state of charge of your house battery in percent is all you really need for this. If you can configure the update intervall, 30s or less is a good starting point.

### 2. Familiarization: 

Get yourself aquainted with the Tesla and PV integrations before you continue. Especially, you should get a feeling on how fast the integrations update and check if your Tesla updates the location and the charging port status when you get home and plug the cable in.

### 3. Sensors: Now, we will create a few sensors and variables that are needed to control the PV current.
   3.1 In Settings/Devices/Helpters, create the following helpers. Keep in mind: My car is called "Tesla" in HA. If yours is Coolest-car-ever, you will want to change the entity names from "xxx.Tesla-xxx" to "xxx.Coolest-car-ever.xxx":

Chose the type "number":

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/House_minimum_SOC.png" width=300>
   
Chose the type "toggle":

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Helper-Hand-Control.png" width=300>

Chose the type "number":

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla_Charge_Break.png" width=300>
   

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla_charge_stop.png" width=300>


   <img src="https://github.com/top-gun/Tesla-PV-charging/assets/3148118/3f154fea-fa0a-4659-b1cd-c74d709cc63a" width=300>

   

   3.2 In Studio Code Server, open the file "configuration.yaml" and add the following sensor. 

   **Attention:**

- The entities sensor.battery_state_of_capacity and sensor.inverter_input_power are specific to my Huawei PV system. If you run a Fronius, SolarEdge, Victron, Sungrow or whatever PV system, you will need to find the right entity names for your system.
- My car is called "Tesla", therefore the entities for my car have "tesla" after the dot. If your cars name is "godzilla", you need to change that to ie sensor.godzilla_charger_power . 
```

  - sensor:
    - name: 'Autocharge-optimal'
      unit_of_measurement: ""
      state: > 
        {% set Battery = states('sensor.battery_state_of_capacity')|float %}
        {% set PV = states('sensor.inverter_input_power')|float -500 %}
        {% set Charge = states('sensor.tesla_charger_power')|float %}
        {% set Throttle = states('input_number.tesla_charge_break') |float %}
        {% set Endoffastcharge = states('input_number.num_battery_min_home') |float %}
        {# PV/230 is the current in a one phase system. For three-phase charging divide by 3! #}
        {% set PVAMP = (PV/230/3) %}
        {# While the house battery is below 90% ist, use only 80% of the output so the house battery will charge, too #}
        {% if Battery<90 %} {% set PVAMP = (PVAMP*0.8) %} {% endif %}
        {# When the house battery is above "Endoffastcharge", use at least 5A. When starting, use +5 for hysteresis #}
        {% if (PVAMP<4) and (Battery>Endoffastcharge+5 )  %}  {% set PVAMP = 6 %} {% endif %} 
        {% if (PVAMP<4) and (Battery>Endoffastcharge) and (Charge > 0) %} {% set PVAMP = 5 %} {% endif %} 
        {# Under 40%, charge only the house battery, not the car. 3% hysteresis: Don't turn on until we reach 43%. #}
        {% if (Battery<43) and (Charge==0) and (PVAMP<6) %} {% set PVAMP = 0 %} {% endif %} 
        {% if (Battery<40) and (Charge>0) %} {% set PVAMP = 0 %} {% endif %} 
        {# Exception: When the PV output is really high, allow charging so we don't feed to the grid early #}
        {% if (PVAMP>6) and (Battery<40) %} {% set PVAMP = 3 %} {% endif %}
        {# Don't start charging under 3A because it's not efficient. #}
        {% if (PVAMP<3) and (Charge==0) %} {% set PVAMP = 0 %} {% endif %}
        {# Under very high load, like cooking at noon, use throttle. Throttle is controlled by an automation #}
        {% set PVAMP = PVAMP - Throttle %}
        {{ PVAMP|int }}
```

### 4. Automations:

We need several automations. Unfortunately, they can grow pretty long, and there is no really good way to export them. So sorry if I can only give you some very huge screenshots. If anybody know a tool for making better visual representations, please let me know.

   4.1 Tesla-Charge-Adjust: This most important one will simply start every 60s and, after checking the car is at home and wired for charging, set the right current and start or stop the charging process.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-Tesla-adjust.png" width=300>

   4.2 Tesla-Leaving: When the car leaves home, set the charge mode to automatic and the house battery to 75% minimum for fast charging.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla-Leaving.png" width=300>

   4.3 Tesla-1800-charge-socket-90p: At 6PM, set the minimum socket for the house battery to 90% so the house battery gets as full as possible

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-1800-socket-90p.png" width=300>

   4.4 Tesla-throttle-high-load: You may run into situations where high house load and high car charging exceed the inverter capacity. My inverter is capable of 11,000W, I set the threshold to 10,500W. Adjust to about 5% below your inverter's capacity.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-throttle-charge-high-load.png" width=300>

   4.5 Tesla-polling-6AM: Start polling the car status at 6:30AM. We stop polling at 11PM to make sure the car can actually sleep.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-turn-on-polling-6AM.png" width=300>

   4.6 Tesla-stop-polling-11PM: Stop polling at 11PM to make sure the car can actually sleep.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-turn-off-polling-11PM.png" width=300>

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
  home:
    entity: hau
  individual1:
    entity: sensor.tesla_charger_power
    name: Auto
    icon: mdi:car
    display_zero: true
    decimals: 2
clickable_entities: true
display_zero_lines:
  mode: show
  transparency: 50
  grey_color:
    - 189
    - 189
    - 189
use_new_flow_rate_model: true
w_decimals: 0
min_flow_rate: 0.75
max_flow_rate: 6
max_expected_power: 2000
min_expected_power: 0.01
watt_threshold: 1000
transparency_zero_lines: 0
kw_decimals: 2
```

   5.2 Mini-Graph-Card: It shows a graph with the car's SOC and the charging power over the last six hours (adjustable).

<img src="[https://github.com/top-gun/Tesla-PV-charging/assets/3148118/6e091030-75ba-48b9-be3b-978b65dd70bb](https://github.com/top-gun/Tesla-PV-charging/assets/3148118/9f1ac8a9-4fd4-4237-a633-8cc4ad5d2291)" width=300>

My configuration:

```
type: custom:mini-graph-card
entities:
  - entity: sensor.tesla_battery
  - entity: sensor.tesla_charger_power
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



   <img src="https://github.com/top-gun/Tesla-PV-charging/assets/3148118/438d8f52-deaf-4521-a13d-162fb1243fd8" width=300>


The definition:

type: entities
entities:
  - entity: binary_sensor.tesla_charging
  - entity: binary_sensor.tesla_charger
    name: Tesla Ladekabel
  - entity: sensor.autocharge_optimal
  - entity: sensor.tesla_charger_power
  - entity: switch.tesla_polling
    name: Allow Tesla polling
  - entity: button.tesla_force_data_update
state_color: true
show_header_toggle: false
title: Tesla Status

   5.4 Tesla control: Information about the charging process, also switches that change to manual charge control (app) or forbid auto charging.

   <img src="https://github.com/top-gun/Tesla-PV-charging/assets/3148118/629f9803-c3bd-4e85-a4f7-ebe033a9d079" width=300>

```
type: conditional
conditions:
  - entity: input_boolean.auto_manuell
    state: 'off'
card:
  type: entities
  entities:
    - entity: number.tesla_charge_limit
    - entity: input_number.num_battery_min_home
    - entity: sensor.tesla_time_charge_complete
    - entity: input_boolean.auto_manuell
    - entity: input_boolean.switch_tesla_no_charge
    - entity: number.tesla_charging_amps
  show_header_toggle: false
  state_color: true
  title: Charge Control
```
