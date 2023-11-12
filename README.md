# esp8266-window-sensor-with-delayed-mail-reminder
After power on an ESP8266 a mail will be sent after a configurable timeout to configurable email recipients as a reminder. Can be used as a window still open reminder.

For saving limited battery power and optimize the battery life the sketch enables the WiFi after the timout. Without WiFi it consumes 20mA. With Wifi it takes 200mA.

If the mail has been sent the ESP goes into deepsleep which consumes some uA.


Parts:

- ESP8266 Board or ESP32 Module
- 2 AA Rechargable Batteries with Holder
- 2.0V-3.0V to 3.3V step up DC/DC Converter
- Reed Switch with magnet
- 10uF/6.3V Capacitor

  See circuit.png

  ![circuit](https://github.com/casu/esp8266-window-sensor-with-delayed-mail-reminder/assets/11190081/0bbb3840-54c4-47fd-a88c-52b3cd6cca26)


Requirements:

- Arduino IDE
- Board Administrator URLs: https://dl.espressif.com/dl/package_esp32_index.json, http://arduino.esp8266.com/stable/package_esp8266com_index.json
- Addition Library: ESP Mail Client by Mobizt

