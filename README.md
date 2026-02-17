# automated-liquid-waste-management-system
detects the blockage in sewage system and makes the alerts 
#include <WiFi.h>
#include <WebServer.h>

// --- 1. WIFI CREDENTIALS ---
// Replace with your actual Wi-Fi name and password
const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";

// --- 2. PIN DEFINITIONS ---
const int trigPinUp   = 5;   // Upstream Trigger
const int echoPinUp   = 18;  // Upstream Echo
const int trigPinDown = 19;  // Downstream Trigger
const int echoPinDown = 21;  // Downstream Echo
const int alertLed    = 13;  // Physical Red LED

// --- 3. SYSTEM CONSTANTS ---
const float emptyThreshold = 70.0; // Distance (cm) when pipe is empty
const float blockageGap    = 40.0; // Difference (cm) to trigger alert

// Start the web server on port 80
WebServer server(80);

// Helper function to calculate distance
float getDistance(int trig, int echo) {
  digitalWrite(trig, LOW);
  delayMicroseconds(2);
  digitalWrite(trig, HIGH);
  delayMicroseconds(10);
  digitalWrite(trig, LOW);
  long duration = pulseIn(echo, HIGH, 30000); // 30ms timeout
  if (duration == 0) return 999; // Error fallback
  return duration * 0.034 / 2;
}

// --- 4. PROMETHEUS METRICS FUNCTION ---
// This function creates the text format Prometheus needs
void handleMetrics() {
  float distUp = getDistance(trigPinUp, echoPinUp);
  float distDown = getDistance(trigPinDown, echoPinDown);

  // Update physical LED logic during the scrape
  if ((distDown - distUp > blockageGap) && (distUp < emptyThreshold)) {
    digitalWrite(alertLed, HIGH);
  } else {
    digitalWrite(alertLed, LOW);
  }

  // Format response for Prometheus
  String m = "# HELP nilas_upstream_cm Upstream distance in cm\n";
  m += "# TYPE nilas_upstream_cm gauge\n";
  m += "nilas_upstream_cm " + String(distUp) + "\n";
  
  m += "# HELP nilas_downstream_cm Downstream distance in cm\n";
  m += "# TYPE nilas_downstream_cm gauge\n";
  m += "nilas_downstream_cm " + String(distDown) + "\n";

  server.send(200, "text/plain", m);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(trigPinUp, OUTPUT);   pinMode(echoPinUp, INPUT);
  pinMode(trigPinDown, OUTPUT); pinMode(echoPinDown, INPUT);
  pinMode(alertLed, OUTPUT);

  // Connect to Wi-Fi
  Serial.print("Connecting to: "); Serial.println(ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500); Serial.print(".");
  }

  Serial.println("\n--- NILAS ONLINE ---");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP()); // COPY THIS IP FOR PROMETHEUS

  // Setup Web Server paths
  server.on("/metrics", handleMetrics);
  server.begin();
}

void loop() {
  server.handleClient(); // Listen for Prometheus requests
}

