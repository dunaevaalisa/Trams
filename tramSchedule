#include <SoftwareSerial.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>
#include <time.h>
#include <map>
#include <string>
#include <MD_Parola.h>
#include <MD_MAX72XX.h>
#include <WebServer.h>
#include "secrets.h"

// Creating a web server instance on port 80
WebServer server(80);

// WiFi and API credentials
const char* ssid = WIFI_SSID;
const char* password = WIFI_PASSWORD;
const char* apiUrl = API_URL;
const char* apiKey = API_KEY;
WiFiClientSecure client;

// 2 LED matrix display settings
#define HARDWARE_TYPE MD_MAX72XX::FC16_HW
#define MAX_DEVICES 8
#define CS_PIN_1 4
#define CS_PIN_2 16

MD_Parola Display1 = MD_Parola(HARDWARE_TYPE, CS_PIN_1, MAX_DEVICES);
MD_Parola Display2 = MD_Parola(HARDWARE_TYPE, CS_PIN_2, MAX_DEVICES);

uint8_t scrollSpeed = 45;
textEffect_t scrollEffect = PA_SCROLL_LEFT;
textPosition_t scrollAlign = PA_LEFT;

// Structure to store stop information
struct StopInfo {
  String stopName;
  String stopId1;
  String stopId2;
};

// Map to hold stop names and their corresponding ids
std::map<String, StopInfo> stationIDMap;

// Function to find stop information based on stop name that is read from curent_stop.txt file on the server
StopInfo findStopInfo(String stopName) {
  auto it = stationIDMap.find(stopName);
  if (it != stationIDMap.end()) {
    return it->second;
  }
  return {"Default", "HSL:12345", "HSL:12346"};
}
// Function to clean special characters from text
String cleanText(String text) {
  text.replace("ä", "a");
  text.replace("ö", "o");
  text.replace("Ä", "A");
  text.replace("Ö", "O");
  return text;
}


void setup() {
  Serial.begin(115200);

  // Initialize both displays
  Display1.begin();
  Display2.begin();
  Display1.setIntensity(10);
  Display2.setIntensity(10);
  Display1.displayClear();
  Display2.displayClear();

  // Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");

  // Synchronize time with NTP server
  const char* ntpServer = "pool.ntp.org";
  const long gmtOffset_sec = 0;
  const int daylightOffset_sec = 10800;
  
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeinfo;

  while (!getLocalTime(&timeinfo)) {
    Serial.println("Waiting for NTP time sync...");
    delay(500);
  }

  Serial.println("Time synchronized!");

  // Populate station id map
  stationIDMap["Pasilan asema"] = {"Pasilan asema", "HSL:1174410", "HSL:1174411"};
  stationIDMap["Töölöntori"] = {"Töölöntori", "HSL:1140447", "HSL:1140449"};
  stationIDMap["Messukeskus"] = {"Messukeskus", "HSL:1173433", ""};
  stationIDMap["Asemapäällikönkatu(7,13)"] = {"Asemapäällikönkatu", "HSL:1173407", "HSL:1173434"};
  stationIDMap["Asemapäällikönkatu"] = {"Asemapäällikönkatu", "HSL:1173401", "HSL:1173402"};
}

// Function to calculate minutes left until departure
int calculateMinutesLeft(int realtimeDeparture) {
  time_t now;
  struct tm timeinfo;
  time(&now);
  localtime_r(&now, &timeinfo);

  long currentSecondsToday = timeinfo.tm_hour * 3600 + timeinfo.tm_min * 60 + timeinfo.tm_sec;
  long timeDifferenceSeconds = realtimeDeparture - currentSecondsToday;

  if (timeDifferenceSeconds < -3600) {
    timeDifferenceSeconds += 24 * 3600;
  }
  long minutesLeft = timeDifferenceSeconds / 60;
  return minutesLeft;
}

// Function to fetch and format departure data for first display
String fetchDataForMatrix1(const String& stopId) {
  HTTPClient http;
  // Build the GraphQL query for the id
  String query = "{\"query\": \"{ stop(id: \\\"" + stopId + "\\\") { name stoptimesWithoutPatterns { realtimeDeparture headsign trip { routeShortName } } } }\"}";

  http.begin(apiUrl);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("digitransit-subscription-key", apiKey);

  int httpCode = http.POST(query);
  if (httpCode > 0) {
    // If the request was successful, parse the response
    String payload = http.getString();
    Serial.println("Response code: " + String(httpCode));
    Serial.println("Response payload: " + payload);
    DynamicJsonDocument doc(2048);
    DeserializationError error = deserializeJson(doc, payload);

    if (error) {
      Serial.print("deserializeJson() failed: ");
      Serial.println(error.c_str());
      return "error";
    }

    JsonArray stoptimes = doc["data"]["stop"]["stoptimesWithoutPatterns"];
    String displayText = "";
    int count = 0;

    // Loop through the upcoming departures (3 entries)
    for (JsonObject stoptime : stoptimes) {
      if (count >= 3) break;

      long realtimeDeparture = stoptime["realtimeDeparture"].as<long>();
      String headsign = stoptime["headsign"].as<String>();
      int spaceIndex = headsign.indexOf(' ');
      String shortHeadsign = (spaceIndex != -1) ? headsign.substring(0, spaceIndex) : headsign;
      String routeShortName = stoptime["trip"]["routeShortName"].as<String>();

      // Calculate minutes left and format the output string
      long minutesLeft = calculateMinutesLeft(realtimeDeparture);
      displayText += cleanText(routeShortName + " " + shortHeadsign + " : " + String(minutesLeft) + " min     " + "     ");
      count++;
    }
    http.end();
    return displayText;

  } else {
    Serial.println("Error on HTTP request");
    http.end();
    return "error";
  }
}

// Function to fetch and format departure data for second display
String fetchDataForMatrix2(const String& stopId) {
  HTTPClient http;
  String query = "{\"query\": \"{ stop(id: \\\"" + stopId + "\\\") { name stoptimesWithoutPatterns { realtimeDeparture headsign trip { routeShortName } } } }\"}";
  http.begin(apiUrl);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("digitransit-subscription-key", apiKey);
  int httpCode = http.POST(query);
  if (httpCode > 0) {
    String payload = http.getString();
    Serial.println("Response code: " + String(httpCode));
    Serial.println("Response payload: " + payload);
    DynamicJsonDocument doc(2048);
    DeserializationError error = deserializeJson(doc, payload);
    if (error) {
      Serial.print("deserializeJson() failed: ");
      Serial.println(error.c_str());
      return "error";
    }
    JsonArray stoptimes = doc["data"]["stop"]["stoptimesWithoutPatterns"];
    String displayText = "";
    int count = 0;

    for (JsonObject stoptime : stoptimes) {
      if (count >= 3) break;

      long realtimeDeparture = stoptime["realtimeDeparture"].as<long>();
      String headsign = stoptime["headsign"].as<String>();
      int spaceIndex = headsign.indexOf(' ');
      String shortHeadsign = (spaceIndex != -1) ? headsign.substring(0, spaceIndex) : headsign;
      String routeShortName = stoptime["trip"]["routeShortName"].as<String>();
      long minutesLeft = calculateMinutesLeft(realtimeDeparture);
  
      displayText += cleanText(routeShortName + " " + shortHeadsign + " : " + String(minutesLeft) + " min     "  + "     ");
      count++;
    }

    http.end();
    return displayText;

  } else {
    Serial.println("Error on HTTP request");
    http.end();
    return "error";
  }
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // Fetch current stop ID from server
    String currentStopId = fetchCurrentStopId();
    StopInfo currentStop = findStopInfo(currentStopId);
    // Fetch and display data for both directions
    String text1 = fetchDataForMatrix1(currentStop.stopId1);
    if (currentStop.stopId2 != "") {
      String text2 = fetchDataForMatrix2(currentStop.stopId2); // Check if there is a corresponding opposite direction
      displayOnMatrix(Display2, text2);
    } else {
      displayOnMatrix(Display2, "One way stop");
    }
    displayOnMatrix(Display1, text1);
    delay(3000); // Short pause before updating again
  }
}

// Function to display text on a matrix
void displayOnMatrix(MD_Parola& display, String text) {
  display.displayText(text.c_str(), scrollAlign, scrollSpeed, 0, scrollEffect, scrollEffect);

  while (!display.displayAnimate()) {}
  display.displayReset();
}

// Function to fetch the current selected stop  from server
String fetchCurrentStopId() {
  HTTPClient http;
  http.begin(SERVER_URL);
  int httpCode = http.GET();

  if (httpCode > 0) {
    String stopId = http.getString();
    stopId.trim();
    http.end();
    return stopId;
    
  } else {
    Serial.println("Error fetching current_stop.txt");
    http.end();
    return "default-stop-id";
  }
}

// Function for deep sleep of the device from 8 pm to 7 am
void checkSleepTime() {
  time_t now = time(nullptr);
  struct tm* timeInfo = localtime(&now);
  int hour = timeInfo->tm_hour;

  if (hour >= 20 || hour < 7) {
    Serial.println("Night time detected. Going to deep sleep...");
    esp_sleep_enable_timer_wakeup(10LL * 60LL * 1000000LL); 
    esp_deep_sleep_start();
  }
}
