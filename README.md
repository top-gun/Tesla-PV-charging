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
   7.1 In Settings/Devices/Helpters, create the following helpers:
   
