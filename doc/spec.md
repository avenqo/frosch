## Spezifikation "Frosch"
Frosch ist eine Reihe von Indoor- und Outdoor-Wettersensoren an verschiedenen Orten innerhalb des selben WLAN-Netzes, die auf Basis von ESPHome unter Verwendung eines ESP8266 /ESP32 plus diverser Komponenten Wetterdaten erfasst und an einen MQTT-Server sendet.  
  
  Sie sollen  
  - Umweltdaten liefern (Sensirion SPS30)
  - Temperatur und Luftdruck mit BMP280 erfassen
  - Ein Regenmelder soll Niederschlag melden  
  - Die Helligkeit messen (Tageslicht)  
  - Ein IR-Empfänger soll Manipulationen erkennen können (Bewegungen in der Nähe)
  
Die Stromversorgung erfolgt zunächst über USB-Steckernetzteile (immer Indoor; wenn notwendig mit Kabeldurchbruch)   
  
Grundsätzlich soll die Möglichkeit bestehen, später andere Sensoren, bspw. einen Windsensor zu integrieren.
 
Gehäuse für Outdoor werden entworfen und gedruckt (PETG).   
 
 ### Einzelsensoren
 #### Indoor: Wohnzimmer
 - ESP8266 NodeMCU V0.9
 - BMP280 für Temperatur und Luftdruck
 - ADS1115 und daran angeschlossen
 	- Fotowiderstand an für Helligkeit
 	- MQ2 Sensor
 - IR Motion Detector HC-SR501
 
#### Outdoor: Norden
 
 - ESP8266 NodeMCU V2
 - BME280 (Temperatur, Luftfeuchtigkeit, Luftdruck)
 - Fotowiderstand an A0 (+3,3V -> LDR -> A0 -> 27k -> GND) für Tagesllicht  
 - SPS30 Feinstaubmessung  
 - Regensensor
 - IR Motion Detector HC-SR501