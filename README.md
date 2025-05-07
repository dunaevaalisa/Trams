# ESP32 Tram Schedule Display

This project displays real-time tram departure times for selected stops in Helsinki using an ESP32 and two LED matrices. It fetches data from the HSL API and updates the information on the displays based on the stop selected through a simple [web interface](https://www.hh3dlab.fi/tram/) that can be opened by scanning the QR-code displayed on OLED screen.

##  Features

- Real-time tram departure display on LED matrices
- Fetches data from HSL’s public transport API
- ESP32 connects to Wi-Fi and updates data every few seconds
- Displays both directions of the selected tram stop (if available)
- Web interface to change the selected stop remotely
- Built-in time synchronization using NTP
- Automatically turns off displays at night to save energy 

## Components Used

- 2 × ESP32 development boards
- 2 × MAX7219-based 8x8 LED matrix displays
- OLED screen 

## How It Works

1. ESP32 connects to Wi-Fi and syncs time using NTP
1.  A PHP web server hosts a file (`current_stop.txt`) containing the selected stop ID
2. ESP32 checks the server for the current stop ID. If it changes, it fetches new tram schedule data using the stop IDs defined in the code
3. The departure times for each direction are displayed on the two LED matrices
4. The device uses the synchronized clock to determine night hours (between 20:00 - 7:00) and powers off the displays during that time, making the project more environmentally friendly and efficient

## API

This project uses HSL's GraphQL API to retrieve tram departure data based on stop IDs. Example stop IDs can be found in [HSL’s documentation or from Digitransit API tools](https://digitransit.fi/en/developers/apis/1-routing-api/)

## Stop id structure and mapping

The structure is built around a stop name, which is fetched from the server. This name is then used to look up the corresponding HSL stop ids in a predefined map called `stationIDMap`. Each entry in the map links a stop name `"Pasilan asema"` to a `StopInfo` struct containing one or two HSL stop ids.

```cpp
struct StopInfo {
  String name;
  String stopId1;
  String stopId2;  // May be empty if the stop is one-directional
};
```

## Future improvements 

- Add support for bus stops or multi-line filtering
- Cache last valid data to show when offline
- Extend web interface with map selection
