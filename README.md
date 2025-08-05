# ESP32-CAM
Experimento con camara esp32 hecho con IA

This sketch will allow you to capture new photos, view the most recent one, and even rotate it, all from a web browser on your smartphone or computer.
Project Overview
This project turns your ESP32-CAM into a mini web server. When you connect to its IP address in your local network, it will serve a web page with the following features:

Of course! As an expert in Arduino sketches and the ESP32-CAM, I can provide you with a comprehensive and well-documented sketch to create a web interface for handling pictures. This sketch will allow you to capture new photos, view the most recent one, and even rotate it, all from a web browser on your smartphone or computer.

### Project Overview

This project turns your ESP32-CAM into a mini web server. When you connect to its IP address in your local network, it will serve a web page with the following features:

*   **Capture Photo:** A button to command the ESP32-CAM to take a new picture.
*   **View Last Photo:** The web page will display the most recently captured photograph.
*   **Rotate Photo:** If the image orientation isn't correct, you can rotate it with the click of a button.

The captured image is saved to the ESP32's built-in SPIFFS (SPI Flash File System), which means the last photo will still be available even after you reboot the board.

### Hardware Requirements

To get started, you will need the following components:

*   An ESP32-CAM board with an OV2640 camera.
*   An FTDI programmer (like a USB to TTL serial converter) to upload the code.
*   Jumper wires for making connections.
*   A 5V power supply, as the FTDI programmer might not provide enough power.

### Wiring the ESP32-CAM for Uploading

Before you can upload the sketch, you need to connect the ESP32-CAM to your computer using the FTDI programmer.

**Important:** For the ESP32-CAM to enter flashing mode, you must connect the `GPIO 0` pin to `GND`.

Here are the typical connections:

| ESP32-CAM Pin | FTDI Programmer Pin |
| :--- | :--- |
| 5V | VCC (Set to 5V) |
| GND | GND |
| U0TXD (GPIO 1) | RX |
| U0RXD (GPIO 3) | TX |

After successfully uploading the sketch, you must disconnect the `GPIO 0` from `GND` and press the reset button to run the program.

### The Arduino Sketch

Here is the complete Arduino sketch. It includes the HTML, CSS, and JavaScript needed for the web interface.

```cpp
/*********
  Rui Santos
  Complete project details at https://RandomNerdTutorials.com/esp32-cam-take-photo-display-web-server/
*********/

#include "WiFi.h"
#include "esp_camera.h"
#include "esp_timer.h"
#include "img_converters.h"
#include "Arduino.h"
#include "soc/soc.h"           // Disable brownour problems
#include "soc/rtc_cntl_reg.h"  // Disable brownour problems
#include "driver/rtc_io.h"
#include <ESPAsyncWebServer.h>
#include <SPIFFS.h>
#include <FS.h>

// Replace with your network credentials
const char* ssid = "REPLACE_WITH_YOUR_SSID";
const char* password = "REPLACE_WITH_YOUR_PASSWORD";

// Create AsyncWebServer object on port 80
AsyncWebServer server(80);

boolean takeNewPhoto = false;

// Photo File Name to save in SPIFFS
#define FILE_PHOTO "/photo.jpg"

// OV2640 camera module pins (CAMERA_MODEL_AI_THINKER)
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27

#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE HTML><html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { text-align:center; }
    .vert { margin-bottom: 10%; }
    .hori{ margin-bottom: 0%; }
  </style>
</head>
<body>
  <div id="container">
    <h2>ESP32-CAM Last Photo</h2>
    <p>It might take more than 5 seconds to capture a photo.</p>
    <p>
      <button onclick="rotatePhoto();">ROTATE</button>
      <button onclick="capturePhoto()">CAPTURE PHOTO</button>
      <button onclick="location.reload();">REFRESH PAGE</button>
    </p>
  </div>
  <div><img src="/photo" id="photo" width="100%"></div>
</body>
<script>
  function capturePhoto() {
    var xhr = new XMLHttpRequest();
    xhr.open('GET', "/capture", true);
    xhr.send();
  }
  
  var deg = 0;
  function rotatePhoto() {
    var img = document.getElementById("photo");
    deg += 90;
    if(isOdd(deg/90)){document.getElementById("container").className = "vert";}
    else{document.getElementById("container").className = "hori";}
    img.style.transform = "rotate(" + deg + "deg)";
  }
  
  function isOdd(n) {
    return Math.abs(n % 2) == 1;
  }
</script>
</html>)rawliteral";

void setup() {
  // Serial port for debugging purposes
  Serial.begin(115200);

  // Disable brownout detector
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);

  // Camera configuration
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  
  // Frame size
  config.frame_size = FRAMESIZE_UXGA;
  config.jpeg_quality = 10;
  config.fb_count = 2;

  // Camera init
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return;
  }

  // Wi-Fi connection
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");

  // Print ESP32-CAM IP Address
  Serial.print("IP Address: http://");
  Serial.println(WiFi.localIP());

  // Initialize SPIFFS
  if(!SPIFFS.begin(true)){
    Serial.println("An Error has occurred while mounting SPIFFS");
    return;
  }

  // Route for root / web page
  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/html", index_html);
  });

  // Route to capture a photo
  server.on("/capture", HTTP_GET, [](AsyncWebServerRequest *request){
    takeNewPhoto = true;
    request->send(200, "text/plain", "Taking photo...");
  });

  // Route to load the photo
  server.on("/photo", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send(SPIFFS, FILE_PHOTO, "image/jpeg");
  });

  // Start server
  server.begin();
}

void loop() {
  if (takeNewPhoto) {
    capturePhotoSaveSpiffs();
    takeNewPhoto = false;
  }
}

// Function to take a photo and save it to SPIFFS
void capturePhotoSaveSpiffs(void) {
  camera_fb_t * fb = NULL; // pointer
  // Take a photo with the camera
  fb = esp_camera_fb_get();  
  if(!fb) {
    Serial.println("Camera capture failed");
    return;
  }
  
  // Photo file path
  File file = SPIFFS.open(FILE_PHOTO, FILE_WRITE);

  // GFX moving it to memory
  if(!file){
    Serial.println("Failed to open file in writing mode");
  } 
  else {
    file.write(fb->buf, fb->len); // payload (image), payload length
    Serial.print("The picture has been saved in ");
    Serial.println(FILE_PHOTO);
  }
  
  // Close the file
  file.close();
  
  // Return the frame buffer back to the driver for reuse
  esp_camera_fb_return(fb);
}
```

### How the Code Works

1.  **Libraries and Definitions:** The code starts by including necessary libraries for WiFi, the camera, and the asynchronous web server. It also defines the camera pins for the AI-Thinker model and your WiFi credentials.
2.  **Web Page (HTML & CSS):** The `index_html` variable stores the HTML and CSS code for the web interface. It's stored in program memory (`PROGMEM`) to save RAM. The page includes buttons that trigger JavaScript functions.
3.  **JavaScript Functions:**
    *   `capturePhoto()`: Sends a GET request to the `/capture` URL on the ESP32-CAM, which sets a flag to take a new photo.
    *   `rotatePhoto()`: Rotates the displayed image by 90 degrees each time the "ROTATE" button is clicked.
4.  **`setup()` Function:**
    *   Initializes the serial communication for debugging.
    *   Configures and initializes the camera with settings like resolution (`FRAMESIZE_UXGA`) and image format (`PIXFORMAT_JPEG`).
    *   Connects to your WiFi network and prints the IP address of the ESP32-CAM.
    *   Initializes the SPIFFS file system.
    *   Sets up the web server routes:
        *   `/`: Serves the main HTML page.
        *   `/capture`: Handles the request to take a new photo.
        *   `/photo`: Sends the saved `photo.jpg` file from SPIFFS to the browser.
5.  **`loop()` Function:** The main loop continuously checks if the `takeNewPhoto` flag is true. If it is, it calls the `capturePhotoSaveSpiffs()` function.
6.  **`capturePhotoSaveSpiffs()` Function:** This function captures an image from the camera, opens a file named `photo.jpg` in SPIFFS, writes the image data to it, and then closes the file.

### Uploading the Sketch and Demonstration

1.  **Prepare Arduino IDE:** Make sure you have the ESP32 board add-on installed in your Arduino IDE. Go to **Tools > Board** and select "AI-Thinker ESP32-CAM".
2.  **Upload:** Connect your ESP32-CAM to the FTDI programmer as described earlier (with GPIO 0 connected to GND). Select the correct COM port and click the upload button. You may need to press the onboard reset button when you see "Connecting........" in the debug window.
3.  **Run:** After a successful upload, disconnect the wire between GPIO 0 and GND. Open the Serial Monitor at a baud rate of 115200 and press the reset button. The ESP32-CAM will connect to your WiFi and print its IP address.
4.  **Access the Web Server:** Open a web browser and enter the IP address shown in the Serial Monitor. You will see the web interface, ready to take and display photos.
