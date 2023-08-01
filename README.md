# Tesla-PV-charging
This documents my solution to charge my Tesla with PV excess. I own a Tesla Model 3, a Huawei PV system with 8.5kW peak and 10kWh battery. The Tesla is plugged into a "dumb" wallbox that can charge with 11kW (3-phase). All charge control is done via the car API, the wallbox is dumb and does just safety stuff. You plug the car into the wallbox, it makes the necessary handshakes to tell the car it's 3-phase 11kW, and turns on the power.

Prerequisites:

You should own a photovoltaic system with enough power to charge your car, with a reasonably sized battery. My system has 8.5kW peak power and a 10kWh battery. You can go lower on the PV power, but it will take longer to charge your car, obviously. The size of the battery is not really important. It will just work as a buffer and even a small 3kWh unit should be just fine. My system is made by Huawei, a Sun2000-10-KTL-M1 with their matching smart-meter and their matching Luna2000 battery. Since I don't use any proprietary functions, it is easy to adapt. I use standard numbers like inverter input power, battery SOC and so on that any standard PV system will provide.

A wallbox (if you charge 3-phase) or the Tesla AC charger that plugs into any ac wallplug. There is no need for the wallbox to have any management capabilities. In fact, this stuff works easier with a wallbox that just charges and doesn't require any commands to do its job.

Then you want Home-Assistant as a house automation system. It is the foundation for what we do here. Home-Assistant has plenty of integrations into the typical house stuff like wall plugs, heating valves and so on. There are integrations for all the common PV brands through a community-managed integration appstore called HACS. You can run Home-Assistant on a Raspberry Pi or similar small computers. The company behind Home-Assistant does also sell ready-to-use systems with their own hardware. If you can find or have a Raspberry Pi at hand, it's cheaper and just fine.

Steps:

1. I assume you have a working Home Assistant installation
2. You need to install HACS (Home Assistant Community Store), it's needed to install the integrations for your PV system and for the Tesla car.
3. You should install "Studio Code Server", it's a comfortable editor the the Home Assistant configuration files. 
4. In HACS, install the Tesla integration and connect it with your Tesla car. There is a good documentation in the HACS page, especially for creating the "tokens" you need to let HA talk with your car.
5. In HACS, install the integration for your PV system and connect it with your local PV system. You don't need all the bells an whistles, the numbers for PV output in Watt and the state of charge of your house battery in percent is all you really need for this. If you can configure the update intervall, 30s or less is a good starting point.
6. Get yourself familiar with the Tesla and PV integrations before you continue. Especially, you should get a feeling on how fast the integrations update and check if your Tesla updates the location and the charging port status when you get home and plug the cable in.

7. Now, we will create a few sensors and variables that are needed to control the PV current.

7.1 In Settings/Devices/Helpters, create the following helpers. Keep in mind: My car is called "Tesla" in HA. If yours is Coolest-car-ever, you will want to change the entity names from "xxx.Tesla-xxx" to "xxx.Coolest-car-ever.xxx":

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/House_minimum_SOC.png" width=300>
   

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Helper-Hand-Control.png" width=300>
   

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla_Charge_Break.png" width=300>
   

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla_charge_stop.png" width=300>

7.2 In Studio Code Server, open the file "configuration.yaml" and add the following sensor. 

Attention: 
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
        {% if (Battery<43) and (Charge==0) %} {% set PVAMP = 0 %} {% endif %} 
        {% if (Battery<40) and (Charge>0) %} {% set PVAMP = 0 %} {% endif %} 
        {# Exception: When the PV output is really high, allow charging so we don't feed to the grid early #}
        {% if (PVAMP>11) and (Battery>20) %} {% set PVAMP = 11 %} {% endif %}
        {# Don't start charging under 3A because it's not efficient. #}
        {% if (PVAMP<3) and (Charge==0) %} {% set PVAMP = 0 %} {% endif %}
        {# Under very high load, like cooking at noon, use throttle. Throttle is controlled by an automation #}
        {% set PVAMP = PVAMP - Throttle %}
        {{ PVAMP|int }}
```

8. Automations:

We need several automations. Unfortunately, they can grow pretty long, and there is no really good way to export them. So sorry if I can only give you some very huge screenshots. If anybody know a tool for making better visual representations, please let me know.

8.1 Tesla-Charge-Adjust: This most important one will simply start every 60s and, after checking the car is at home and wired for charging, set the right current and start or stop the charging process.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-Tesla-adjust.png" width=300>

8.2 Tesla-Leaving: When the car leaves home, set the charge mode to automatic and the house battery to 75% minimum for fast charging.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Tesla-Leaving.png" width=300>

8.3 Tesla-1800-charge-socket-90p: At 6PM, set the minimum socket for the house battery to 90% so the house battery gets as full as possible

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-1800-socket-90p.png" width=300>

8.4 Tesla-throttle-high-load: You may run into situations where high house load and high car charging exceed the inverter capacity. My inverter is capable of 11,000W, I set the threshold to 10,500W. Adjust to about 5% below your inverter's capacity.

   <img src="https://github.com/top-gun/Tesla-PV-charging/blob/main/pictures/Automation-throttle-charge-high-load.png" width=300>
