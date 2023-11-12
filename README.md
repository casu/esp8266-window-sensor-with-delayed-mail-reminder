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

Source Code:

/*
  Carsten Subbe

  When power on the ESP8266 starts blinking the LED connected to D0 every 20 seconds to show that its activated.
  After a 20 minutes timeout we connect to the configured WLAN and send a mail to  given recipients.

  Example adapted from: https://github.com/mobizt/ESP-Mail-Client
*/

#include <Arduino.h>
#if defined(ESP32)
#include <WiFi.h>
#elif defined(ESP8266)
#include <ESP8266WiFi.h>
#endif
#include <ESP_Mail_Client.h>
#define WIFI_SSID "YOUR WIFI SSID"                 // Add your WLAN WWID here
#define WIFI_PASSWORD "YOUR WIFI PASSWORD"         // Add your WLAN password here
/** The smtp host name e.g. smtp.gmail.com for GMail or smtp.office365.com for Outlook or smtp.mail.yahoo.com or mail.arcor.de */
#define SMTP_HOST "YOUR MAIL PROVIDERS SMTP HOST"  // SMTP host of your mail provider
#define SMTP_PORT 465                              // SMTP port
/* The sign in credentials */
#define AUTHOR_EMAIL "YOUR USER NAME FOR THE MAIL SERVER"     // login name of your mail account
#define AUTHOR_PASSWORD "YOUR PASSWORD OF THE MAIL SERVER"    // password of you account
/* Recipient's email*/
#define RECIPIENT_EMAIL1 "mail.1@example.com"
#define RECIPIENT_EMAIL2 "mail.2@example.com"
#define RECIPIENT_EMAIL3 "mail.3@example.com"
#define RECIPIENT_EMAIL4 "mail.4@example.com"
#define RECIPIENT_EMAIL5 "mail.5@example.com"
/* Declare the global used SMTPSession object for SMTP transport */
SMTPSession smtp;
/* Callback function to get the Email sending status */
void smtpCallback(SMTP_Status status);
/* Set the timer values */
long myTimeout = 1200000;           // seconds before sending mail
long MailCanBeSent = 0;             // timer expired = 1
long MailHasBeenSent = 0;           // mail sended = 1


void setup() {
  Serial.begin(115200);
  Serial.println("\n\nWindow opened. ESP wakeup.\nAfter 20 minutes mails will be sent to");
  Serial.println(RECIPIENT_EMAIL1);
  Serial.println(RECIPIENT_EMAIL2);
  Serial.println(RECIPIENT_EMAIL3);
  Serial.println(RECIPIENT_EMAIL4);
  Serial.println(RECIPIENT_EMAIL5);
  Serial.println("if the window is still opened.\n");
  delay(500);
  /*  Set the network reconnection option */
  MailClient.networkReconnect(true);
  /** Enable the debug via Serial port
   * 0 for no debugging
   * 1 for basic level debugging
   *
   * Debug port can be changed via ESP_MAIL_DEFAULT_DEBUG_PORT in ESP_Mail_FS.h
   */
  smtp.debug(1);
  /* Set the callback function to get the sending results */
  smtp.callback(smtpCallback);
  /* Configure builtin LED */
  pinMode(LED_BUILTIN, OUTPUT);     // Define LED as output
  digitalWrite(LED_BUILTIN, HIGH);  // Switch off LED
}


void loop() {
  /* Short LED flash every 20 seconds */
  delay(19100);                     // wait
  digitalWrite(LED_BUILTIN, LOW);   // switch on LED
  delay(100);                       // wait
  digitalWrite(LED_BUILTIN, HIGH);  // switch off LED
  /* Take a look of we exceeded the timeout */
  if (millis() > myTimeout) {
    Serial.println("\nTimeout: Mail can be sent.");  // Timeout reached. Send mail.
    MailCanBeSent = 1;
  } else {
    Serial.print("Timeout: ");
    Serial.print(millis());
    Serial.print(" / ");
    Serial.println(myTimeout);
  }

  if (MailCanBeSent == 1) {
    /* Declare the Session_Config for user defined session credentials */
    Session_Config config;
    /* Set the session config */
    config.server.host_name = SMTP_HOST;
    config.server.port = SMTP_PORT;
    config.login.email = AUTHOR_EMAIL;
    config.login.password = AUTHOR_PASSWORD;
    config.login.user_domain = "";
    /*
    Set the NTP config time
    For times east of the Prime Meridian use 0-12
    For times west of the Prime Meridian add 12 to the offset.
    Ex. American/Denver GMT would be -6. 6 + 12 = 18
    See https://en.wikipedia.org/wiki/Time_zone for a list of the GMT/UTC timezone offsets
    */
    config.time.ntp_server = F("pool.ntp.org,time.nist.gov");
    config.time.gmt_offset = 1;
    config.time.day_light_offset = 0;
    /* Declare the message class */
    SMTP_Message message;
    /* Set the message headers */
    message.sender.name = F("MailSensor");
    message.sender.email = AUTHOR_EMAIL;
    message.subject = F("YOUR MAIL SUBJECT");
    message.addRecipient(F("YOUR SENDER NAME"), RECIPIENT_EMAIL1);
    message.addCc(RECIPIENT_EMAIL2);
    message.addCc(RECIPIENT_EMAIL3);
    message.addCc(RECIPIENT_EMAIL4);
    message.addCc(RECIPIENT_EMAIL5);
        
    String textMsg = "YOUR MAIL TEXT\n\n.Hello World.\n\nBest regards\n  YOUR SENSOR";
    //Send raw text message
    message.text.content = textMsg.c_str();
    message.text.charSet = "us-ascii";
    message.text.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
    message.priority = esp_mail_smtp_priority::esp_mail_smtp_priority_low;
    message.response.notify = esp_mail_smtp_notify_success | esp_mail_smtp_notify_failure | esp_mail_smtp_notify_delay;

    /*Send HTML message*/
    /*String htmlMsg = "<div style=\"color:#2f4468;\"><h1>Hello World!</h1><p>- Sent from ESP board</p></div>";
    message.html.content = htmlMsg.c_str();
    message.html.content = htmlMsg.c_str();
    message.text.charSet = "us-ascii";
    message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit;*/

    /* Connect to WiFi */
    digitalWrite(LED_BUILTIN, LOW);  // Switch on LED

    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    Serial.print("Connecting to Wi-Fi");
    while (WiFi.status() != WL_CONNECTED) {
      Serial.print(".");
      delay(300);
    }
    Serial.println();
    Serial.print("Connected with IP: ");
    Serial.println(WiFi.localIP());
    Serial.println();
    delay(2000);

    /* Start sending Email and close the session */
    MailCanBeSent = 0;
    MailHasBeenSent = 1;
    //Send raw text message
    message.text.content = textMsg.c_str();
    message.text.charSet = "us-ascii";
    message.text.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
    message.priority = esp_mail_smtp_priority::esp_mail_smtp_priority_low;
    message.response.notify = esp_mail_smtp_notify_success | esp_mail_smtp_notify_failure | esp_mail_smtp_notify_delay;
    /* Connect to the server */
    if (!smtp.connect(&config)) {
      ESP_MAIL_PRINTF("Connection error, Status Code: %d, Error Code: %d, Reason: %s", smtp.statusCode(), smtp.errorCode(), smtp.errorReason().c_str());
      return;
    }
    if (!smtp.isLoggedIn()) {
      Serial.println("\nNot yet logged in.");
    } else {
      if (smtp.isAuthenticated())
        Serial.println("\nSuccessfully logged in.");
      else
        Serial.println("\nConnected with no Auth.");
    }
    if (!MailClient.sendMail(&smtp, &message)) {
      ESP_MAIL_PRINTF("Error, Status Code: %d, Error Code: %d, Reason: %s", smtp.statusCode(), smtp.errorCode(), smtp.errorReason().c_str());
    }
  }
  /* Mail has been sent. We go in endless deep sleep and wait for the next power on. */
  if (MailHasBeenSent == 1) {
    Serial.println("Going into deep sleep now. See you again at next power on.");
    delay(1000);
    digitalWrite(LED_BUILTIN, HIGH);  // LED ausschalten
    ESP.deepSleep(0);
  }
}


/* Callback function to get the Email sending status */
void smtpCallback(SMTP_Status status)
  /* Print the current status */
  Serial.println(status.info());
  /* Print the sending result */
  if (status.success()) {
    // ESP_MAIL_PRINTF used in the examples is for format printing via debug Serial port
    // that works for all supported Arduino platform SDKs e.g. AVR, SAMD, ESP32 and ESP8266.
    // In ESP8266 and ESP32, you can use Serial.printf directly.
    Serial.println("----------------");
    ESP_MAIL_PRINTF("Message sent success: %d\n", status.completedCount());
    ESP_MAIL_PRINTF("Message sent failed: %d\n", status.failedCount());
    Serial.println("----------------\n");
    for (size_t i = 0; i < smtp.sendingResult.size(); i++) {
      /* Get the result item */
      SMTP_Result result = smtp.sendingResult.getItem(i);
      // In case, ESP32, ESP8266 and SAMD device, the timestamp get from result.timestamp should be valid if
      // your device time was synched with NTP server.
      // Other devices may show invalid timestamp as the device time was not set i.e. it will show Jan 1, 1970.
      // You can call smtp.setSystemTime(xxx) to set device time manually. Where xxx is timestamp (seconds since Jan 1, 1970)
      ESP_MAIL_PRINTF("Message No: %d\n", i + 1);
      ESP_MAIL_PRINTF("Status: %s\n", result.completed ? "success" : "failed");
      ESP_MAIL_PRINTF("Date/Time: %s\n", MailClient.Time.getDateTimeString(result.timestamp, "%B %d, %Y %H:%M:%S").c_str());
      ESP_MAIL_PRINTF("Recipient: %s\n", result.recipients.c_str());
      ESP_MAIL_PRINTF("Subject: %s\n", result.subject.c_str());
    }
    Serial.println("----------------\n");
    // You need to clear sending result as the memory usage will grow up.
    smtp.sendingResult.clear();
  }
}


