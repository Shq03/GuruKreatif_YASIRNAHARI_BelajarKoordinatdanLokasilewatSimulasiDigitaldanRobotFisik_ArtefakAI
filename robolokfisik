
#include "esp_camera.h"
#include "esp_http_server.h"
#include <WiFi.h>
#include <WebServer.h>

// =================================
// ROBOLOK - ESP32 CAM AI THINKER
// Firmware dasar versi 1.0
// =================================

// WiFi Access Point
const char* ssid = "ROBOLOK";
const char* password = "robolok123";

WebServer server(80);
httpd_handle_t cameraServer = NULL;

// PIN MOTOR - PEMETAAN SEMENTARA
#define IN1  2
#define IN2  12
#define IN3  13
#define IN4  14

// Waktu gerak maksimum satu perintah
const int MAX_MOVE_MS = 2000;

unsigned long motorStopAt = 0;
bool motorActive = false;

// =================================
// KAMERA AI THINKER
// =================================

void startCamera() {
  camera_config_t config;

  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;

  config.pin_d0 = 5;
  config.pin_d1 = 18;
  config.pin_d2 = 19;
  config.pin_d3 = 21;
  config.pin_d4 = 36;
  config.pin_d5 = 39;
  config.pin_d6 = 34;
  config.pin_d7 = 35;

  config.pin_xclk = 0;
  config.pin_pclk = 22;
  config.pin_vsync = 25;
  config.pin_href = 23;

  config.pin_sscb_sda = 26;
  config.pin_sscb_scl = 27;

  config.pin_pwdn = 32;
  config.pin_reset = -1;

  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  config.frame_size = FRAMESIZE_QVGA;
  config.jpeg_quality = 12;
  config.fb_count = 1;

  esp_err_t err = esp_camera_init(&config);

  if (err != ESP_OK) {
    Serial.printf("Camera error: 0x%x\n", err);
    return;
  }

  Serial.println("Camera berhasil aktif");
}

// =================================
// STREAMING SERVER PORT 81
// =================================

static esp_err_t streamHandler(httpd_req_t *req) {

  esp_err_t res = httpd_resp_set_type(
    req,
    "multipart/x-mixed-replace;boundary=frame"
  );

  if (res != ESP_OK) return res;

  char header[64];

  while (true) {

    camera_fb_t *fb = esp_camera_fb_get();

    if (!fb) {
      return ESP_FAIL;
    }

    snprintf(
      header,
      sizeof(header),
      "--frame\r\nContent-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n",
      (unsigned int)fb->len
    );

    res = httpd_resp_send_chunk(
      req, header, strlen(header)
    );

    if (res == ESP_OK) {
      res = httpd_resp_send_chunk(
        req, (const char*)fb->buf, fb->len
      );
    }

    if (res == ESP_OK) {
      res = httpd_resp_send_chunk(req, "\r\n", 2);
    }

    esp_camera_fb_return(fb);

    if (res != ESP_OK) break;

    delay(30);
  }

  return res;
}

void startStreamServer() {

  httpd_config_t config = HTTPD_DEFAULT_CONFIG();

  config.server_port = 81;
  config.ctrl_port = 32769;

  httpd_uri_t streamUri = {
    .uri = "/stream",
    .method = HTTP_GET,
    .handler = streamHandler,
    .user_ctx = NULL
  };

  if (httpd_start(&cameraServer, &config) == ESP_OK) {
    httpd_register_uri_handler(cameraServer, &streamUri);
    Serial.println("Streaming aktif pada port 81");
  }
}

// =================================
// MOTOR CONTROLLER
// =================================

void motorStop() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  motorActive = false;
  motorStopAt = 0;
}

void motorForward() {

  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void motorBackward() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);

  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void motorRight() {

  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void motorLeft() {

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);

  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

// =================================
// API PERINTAH ROBOT
// =================================

void handleMove() {

  if (!server.hasArg("d")) {
    server.send(400, "text/plain", "Direction missing");
    return;
  }

  String direction = server.arg("d");
  int duration = 500;

  if (server.hasArg("t")) {
    duration = server.arg("t").toInt();
  }

  duration = constrain(duration, 100, MAX_MOVE_MS);

  // Hentikan perintah sebelumnya
  motorStop();

  if (direction == "F") {
    motorForward();
  }
  else if (direction == "B") {
    motorBackward();
  }
  else if (direction == "R") {
    motorRight();
  }
  else if (direction == "L") {
    motorLeft();
  }
  else {
    server.send(400, "text/plain", "Invalid command");
    return;
  }

  motorActive = true;
  motorStopAt = millis() + duration;

  server.send(200, "text/plain", "Command accepted");
}

void handleStop() {

  motorStop();

  server.send(200, "text/plain", "STOP");
}

// =================================
// WEB CONTROLLER
// =================================

const char webpage[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>RoboLok Controller</title>

<style>
body {
  font-family: Arial;
  background: #eef3f9;
  text-align: center;
  margin: 15px;
}

.card {
  background: white;
  padding: 18px;
  border-radius: 15px;
  max-width: 450px;
  margin: auto;
}

button {
  padding: 18px;
  margin: 5px;
  width: 105px;
  border: 0;
  border-radius: 10px;
  background: #1769aa;
  color: white;
  font-size: 17px;
}

.stop {
  background: #d83232;
}

img {
  width: 100%;
  border-radius: 10px;
}

input {
  width: 75px;
  padding: 8px;
}
</style>
</head>

<body>

<div class="card">

<h2>ROBOLOK</h2>
<p>IoT Robot Learning Controller</p>

<img src="http://192.168.4.1:81/stream">

<h3>Kontrol Robot</h3>

<p>
Durasi gerak (ms):
<input id="duration" type="number"
value="500" min="100" max="2000">
</p>

<div>
<button onclick="move('F')">MAJU</button>
</div>

<div>
<button onclick="move('L')">KIRI</button>
<button class="stop" onclick="stopRobot()">STOP</button>
<button onclick="move('R')">KANAN</button>
</div>

<div>
<button onclick="move('B')">MUNDUR</button>
</div>

<p id="status">Robot siap</p>

</div>

<script>

async function move(direction) {

  let duration = document.getElementById(
    'duration'
  ).value;

  document.getElementById('status').innerText =
    'Menjalankan perintah...';

  try {
    let response = await fetch(
      '/api/move?d=' + direction +
      '&t=' + duration
    );

    document.getElementById('status').innerText =
      await response.text();

  } catch(e) {
    document.getElementById('status').innerText =
      'Koneksi gagal';
  }
}

async function stopRobot() {

  await fetch('/api/stop');

  document.getElementById('status').innerText =
    'Robot berhenti';
}

</script>

</body>
</html>
)rawliteral";

void handleRoot() {
  server.send_P(200, "text/html", webpage);
}

// =================================
// SETUP
// =================================

void setup() {

  Serial.begin(115200);

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  motorStop();

  Serial.println("Memulai RoboLok...");

  startCamera();

  WiFi.mode(WIFI_AP);
  WiFi.softAP(ssid, password);

  Serial.print("Alamat robot: ");
  Serial.println(WiFi.softAPIP());

  server.on("/", handleRoot);
  server.on("/api/move", handleMove);
  server.on("/api/stop", handleStop);

  server.begin();

  startStreamServer();

  Serial.println("ROBOLOK SIAP");
}

// =================================
// LOOP
// =================================

void loop() {

  server.handleClient();

  // Penghentian otomatis berdasarkan durasi
  if (
    motorActive &&
    (long)(millis() - motorStopAt) >= 0
  ) {
    motorStop();
  }

  delay(2);
}
