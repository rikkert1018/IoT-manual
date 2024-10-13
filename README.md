<h1>Smart Space Heater with ESP8266 and Telegram</h1>

<p>
This project allows you to control a space heater remotely via Telegram or manually via a button, using an ESP8266 microcontroller and a WS2812B LED strip. It features both automatic and manual modes to turn the heater on or off depending on temperature data fetched from OpenWeatherMap.
</p>

<hr>

<h2>🛠️ Requirements</h2>

<h3>Hardware</h3>
<ul>
  <li><b>ESP8266 Microcontroller</b>: Choose either a NodeMCU or a Wemos D1 Mini.</li>
  <li><b>WS2812B RGB LED Strip</b>: A LED strip that will show the status of the system.</li>
  <li><b>Pushbutton</b>: For selecting between manual and automatic modes.</li>
  <li><b>Jumper Wires</b>: For making electrical connections.</li>
  <li><b>USB Cable</b>: To connect the ESP8266 to your computer.</li>
</ul>

<h3>Software</h3>
<ul>
  <li><b><a href="https://www.arduino.cc/en/software">Arduino IDE</a></b>: The software used to program the ESP8266.</li>
  <li><b>Required Libraries</b>: You will need to install the following libraries within the Arduino IDE:
    <ul>
      <li>ESP8266WiFi: For WiFi connectivity.</li>
      <li>WiFiClientSecure: For secure client connections.</li>
      <li>UniversalTelegramBot: To enable communication with Telegram.</li>
      <li>Adafruit NeoPixel: To control the WS2812B LED strip.</li>
    </ul>
  </li>
</ul>

<hr>

<h2>⚙️ Step 1: Arduino IDE Setup</h2>

<h3>1. Install ESP8266 Support</h3>
<p>
  To add support for the ESP8266 in Arduino IDE, follow these steps:
  <ol>
    <li>Open the Arduino IDE.</li>
    <li>Go to <b>File > Preferences</b>.</li>
    <li>Find the field labeled <b>Additional Board Manager URLs</b> and paste the following URL:</li>
    <code>http://arduino.esp8266.com/stable/package_esp8266com_index.json</code>
    <li>Click <b>OK</b> to close the Preferences window.</li>
    <li>Next, navigate to <b>Tools > Board > Board Manager</b>.</li>
    <li>In the search box, type <b>ESP8266</b> and click the <b>Install</b> button for the ESP8266 by ESP8266 Community.</li>
  </ol>
</p>

<h3>2. Install Libraries</h3>
<p>
  To install the required libraries, follow these steps:
  <ol>
    <li>In the Arduino IDE, go to <b>Sketch > Include Library > Manage Libraries</b>.</li>
    <li>In the Library Manager window, use the search bar to find and install the following libraries:
      <ul>
        <li>ESP8266WiFi</li>
        <li>WiFiClientSecure</li>
        <li>UniversalTelegramBot</li>
        <li>Adafruit NeoPixel</li>
      </ul>
    </li>
  </ol>
</p>

<hr>

<h2>🔌 Step 2: Hardware Connections</h2>
<p>Follow these steps to connect the hardware components:</p>
<ol>
  <li><b>LED Strip</b>: Connect the LED strip’s data pin (DIN) to <b>D2</b> on the ESP8266. Connect the LED strip’s power and ground pins to <b>5V</b> and <b>GND</b> on the ESP8266.</li>
  <li><b>Pushbutton</b>: Connect one side of the button to <b>D1</b> on the ESP8266, and the other side to <b>GND</b>.</li>
</ol>

<hr>

<h2>📝 Step 3: Configure and Upload Code</h2>
<p>To upload the code to the ESP8266, follow these steps:</p>
<ol>
  <li>Open the Arduino IDE and paste the code provided below into a new sketch.</li>
  <li>Replace the placeholder values for your WiFi credentials, OpenWeatherMap API key, and Telegram Bot Token with your own.</li>
</ol>

<pre>
<code>
#include &lt;ESP8266WiFi.h&gt;
#include &lt;WiFiClientSecure.h&gt;
#include &lt;UniversalTelegramBot.h&gt;
#include &lt;Adafruit_NeoPixel.h&gt;

// WiFi settings
#define WIFI_SSID "your_wifi_ssid"
#define WIFI_PASSWORD "your_wifi_password"

// OpenWeatherMap API
const char* server = "api.openweathermap.org";
String city = "Amsterdam,NL";        
String apiKey = "your_api_key";      

// Telegram BOT Token (obtained via Botfather)
#define BOT_TOKEN "your_bot_token"
const unsigned long BOT_MTBS = 5000; 

X509List cert(TELEGRAM_CERTIFICATE_ROOT);
WiFiClientSecure secured_client;
UniversalTelegramBot bot(BOT_TOKEN, secured_client);
WiFiClient client;

unsigned long bot_lasttime = 0; 

// LED strip settings
#define LED_STRIP_PIN D2               
#define NUM_LEDS 8                     
Adafruit_NeoPixel strip(NUM_LEDS, LED_STRIP_PIN, NEO_GRB + NEO_KHZ800);

// Button settings
#define BUTTON_PIN D1                  
bool autoMode = false;                
bool lastButtonState = HIGH;          
bool ledState = false;                 

// Weather update settings
unsigned long lastWeatherUpdate = 0;  
const unsigned long weatherInterval = 10 * 60 * 1000; 
float tempThreshold = 20.0;            

void setup() {
  Serial.begin(115200);
  Serial.println();

  // Connect to WiFi
  Serial.print("Connecting to ");
  Serial.print(WIFI_SSID);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  secured_client.setTrustAnchors(&cert);

  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.print("\nWiFi connected. IP address: ");
  Serial.println(WiFi.localIP());

  // LED and button setup
  strip.begin();
  strip.show();

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Set time via NTP
  configTime(0, 0, "pool.ntp.org");
  time_t now = time(nullptr);
  while (now &lt; 24 * 3600) {
    delay(100);
    now = time(nullptr);
  }
}

void loop() {
  checkButton();

  if (millis() - bot_lasttime &gt; BOT_MTBS) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    while (numNewMessages) {
      handleNewMessages(numNewMessages);
      numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    }
    bot_lasttime = millis();
  }

  if (autoMode &amp;&amp; millis() - lastWeatherUpdate &gt; weatherInterval) {
    getWeatherData();
    lastWeatherUpdate = millis();
  }
}

void getWeatherData() {
  if (client.connect(server, 80)) {
    Serial.println("Connected to weather server");
    client.println("GET /data/2.5/weather?q=" + city + "&appid=" + apiKey + "&units=metric HTTP/1.1");
    client.println("Host: api.openweathermap.org");
    client.println("Connection: close");
    client.println();

    unsigned long timeout = millis();
    while (client.available() == 0) {
      if (millis() - timeout &gt; 5000) {
        Serial.println(">>> Client Timeout!");
        client.stop();
        return;
      }
    }

    String line;
    while (client.available()) {
      line = client.readStringUntil('\n');
      if (line.startsWith("{")) {
        parseWeatherData(line);
        break;
      }
    }
  } else {
    Serial.println("Connection to weather server failed");
  }
}

void parseWeatherData(String jsonString) {
  DynamicJsonDocument doc(1024);
  DeserializationError error = deserializeJson(doc, jsonString);

  if (error) {
    Serial.print("JSON parsing failed: ");
    Serial.println(error.c_str());
    return;
  }

  float temperature = doc["main"]["temp"];
  Serial.print("Current temperature: ");
  Serial.println(temperature);

  if (autoMode &amp;&amp; temperature &lt; tempThreshold) {
    setStripColor(255, 0, 0); 
    ledState = true;
    Serial.println("LED on (cold)");
  } else if (autoMode) {
    setStripColor(0, 0, 0);
    ledState = false;
    Serial.println("LED off (warm)");
  }
}

void setStripColor(uint8_t r, uint8_t g, uint8_t b) {
  for (int i = 0; i &lt; strip.numPixels(); i++) {
    strip.setPixelColor(i, strip.Color(r, g, b));
  }
  strip.show();
}

void handleNewMessages(int numNewMessages) {
  for (int i = 0; i &lt; numNewMessages; i++) {
    String chat_id = String(bot.messages[i].chat_id);
    String text = bot.messages[i].text;

    if (text == "/start") {
      bot.sendMessage(chat_id, "Welcome! Use /manual to switch to manual mode or /auto to switch to automatic mode.");
    } else if (text == "/manual") {
      autoMode = false;
      bot.sendMessage(chat_id, "Manual mode activated. Use /on to turn on the heater and /off to turn it off.");
    } else if (text == "/auto") {
      autoMode = true;
      bot.sendMessage(chat_id, "Automatic mode activated. The heater will turn on/off based on temperature.");
    } else if (text == "/on") {
      ledState = true;
      setStripColor(255, 0, 0);
      bot.sendMessage(chat_id, "Heater turned on.");
    } else if (text == "/off") {
      ledState = false;
      setStripColor(0, 0, 0);
      bot.sendMessage(chat_id, "Heater turned off.");
    }
  }
}

void checkButton() {
  bool currentState = digitalRead(BUTTON_PIN);
  if (currentState == LOW &amp;&amp; lastButtonState == HIGH) {
    autoMode = !autoMode;
    if (autoMode) {
      Serial.println("Automatic mode activated");
    } else {
      Serial.println("Manual mode activated");
    }
  }
  lastButtonState = currentState;
}
</code>
</pre>

<p>3. Select your board from <b>Tools > Board</b>.</p>
<p>4. Select the correct COM port from <b>Tools > Port</b>.</p>
<p>5. Click on the upload button (right arrow icon) to upload the code to the ESP8266.</p>

<hr>

<h2>📱 Step 4: Set Up Your Telegram Bot</h2>
<p>To set up your Telegram bot:</p>
<ol>
  <li>Open Telegram and search for <b>BotFather</b>.</li>
  <li>Start a chat with BotFather and use the command <b>/newbot</b> to create a new bot.</li>
  <li>Follow the instructions to get your <b>Bot Token</b>.</li>
</ol>

<hr>

<h2>✅ Step 5: Enjoy Your Smart Space Heater!</h2>
<p>Once everything is set up, you can control your space heater through Telegram using the following commands:</p>
<ul>
  <li><b>/start</b>: Start the bot.</li>
  <li><b>/manual</b>: Switch to manual mode.</li>
  <li><b>/auto</b>: Switch to automatic mode.</li>
  <li><b>/on</b>: Turn on the heater.</li>
  <li><b>/off</b>: Turn off the heater.</li>
</ul>

<p>Enjoy your smart space heater!</p>
