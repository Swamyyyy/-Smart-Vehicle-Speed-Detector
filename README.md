# Car Speed Detector (Arduino Nano)

A beginner-friendly electronics project that measures the speed of a toy or RC car passing through two sensor gates and shows the result on a 16x2 LCD. LEDs and a buzzer show whether the car is driving at a normal speed, needs caution, or is overspeeding.

## How it works

Two IR obstacle sensors are placed a known distance apart on either side of a track. When the car breaks the first sensor's beam, the Arduino starts a timer, and when it reaches the second sensor, it stops the timer.

```
speed = distance between sensors / time taken
```

The result is multiplied by a `SCALE` value so that a small toy car can be shown as a "scaled" speed (for example, a 1:20 model shown at the speed a full-size car would be doing). Set `SCALE = 1` to see the real speed.

> **Note:** If you use a scale multiplier, label the display or your demo as "scaled speed" so viewers know it is a model-scale reading.

## Features

- Measures speed with two IR sensor gates
- Shows speed and status on a 16x2 I2C LCD
- Green, yellow, and red LEDs that match the speed range
- Buzzer and flashing red LED on overspeed
- Shows "No car detected" after 5 seconds of no activity
- Adjustable gap, scale, speed limits, and display times

## Behaviour

| Speed | LCD line 1 | LCD line 2 | LED | Buzzer | Shown for |
|---|---|---|---|---|---|
| Below 50 km/h | Speed: 32 km/h | Normal Speed | Green | Off | 3 sec |
| 50 to 60 km/h | Speed: 55 km/h | Caution | Yellow | Off | 3 sec |
| Above 60 km/h | Speed: 74 km/h | OVERSPEEDING! | Red (flashing) | Beeping | 4 sec |
| No car for 5 sec | No car detected | | None | Off | Until next car |

After each result, the display returns to "Waiting for car".

## Components

| Part | Qty | Notes |
|---|---|---|
| Arduino Nano | 1 | Header pins soldered on |
| Breadboard | 1 | Half size is enough |
| 16x2 LCD with I2C backpack | 1 | 4 pins: GND, VCC, SDA, SCL |
| IR obstacle sensor module (FC-51 or similar) | 2 | Short range |
| Green LED | 1 | |
| Yellow LED | 1 | |
| Red LED | 1 | |
| 220 ohm resistor | 3 | One per LED |
| Buzzer | 1 | Active buzzer is simplest |
| Jumper wires | About 30 | |
| USB cable | 1 | Match your Nano's connector |
| Toy or RC car | 1 | White tape on its side helps detection |
| Cardboard / foam board | | For the track and gates |

## Wiring

### Power rails

| From | To |
|---|---|
| Nano 5V | Breadboard + rail |
| Nano GND | Breadboard - rail |

### IR sensors

| Sensor pin | Connects to |
|---|---|
| IR 1 VCC | 5V rail |
| IR 1 GND | GND rail |
| IR 1 OUT | D2 |
| IR 2 VCC | 5V rail |
| IR 2 GND | GND rail |
| IR 2 OUT | D3 |

IR 1 is the gate the car reaches first, and IR 2 is the second gate.

### LCD (I2C)

| LCD pin | Connects to |
|---|---|
| GND | GND rail |
| VCC | 5V rail |
| SDA | A4 |
| SCL | A5 |

### LEDs (each through a 220 ohm resistor)

| LED | Connection |
|---|---|
| Green | D9, then resistor, then LED long leg (+). Short leg (-) to GND rail |
| Red | D10, then resistor, then LED long leg (+). Short leg (-) to GND rail |
| Yellow | D11, then resistor, then LED long leg (+). Short leg (-) to GND rail |

### Buzzer

| Buzzer pin | Connects to |
|---|---|
| + (marked or longer leg) | D8 |
| - | GND rail |

### Nano pin summary

| Nano pin | Goes to |
|---|---|
| D2 | IR 1 OUT |
| D3 | IR 2 OUT |
| D8 | Buzzer + |
| D9 | Green LED (via resistor) |
| D10 | Red LED (via resistor) |
| D11 | Yellow LED (via resistor) |
| A4 | LCD SDA |
| A5 | LCD SCL |
| 5V | 5V rail |
| GND | GND rail |

## Gate layout

```
   IR 1        IR 2
    |           |
    |<--15 cm-->|        car travels this way ->
 ===[ track ]======================
```

- Each sensor faces across the track toward the car's path.
- Mount both sensors at the same height as the car's side, 2 to 5 cm from it.
- Measure the gap from the centre of one sensor to the centre of the other, and put that value in `DIST_M`.
- Put white tape on the car's side so the IR sensors detect it reliably.
- If the two sensors trigger each other, put a small piece of cardboard between them or angle them slightly apart.

## Software setup

1. Install the [Arduino IDE](https://www.arduino.cc/en/software).
2. Install the **LiquidCrystal I2C** library from Tools, Manage Libraries.
3. Select **Tools, Board, Arduino Nano**. If the upload fails, try **Processor: ATmega328P (Old Bootloader)**.
4. Copy the code below into a new sketch and upload it.

## Configuration

| Setting | Default | Meaning |
|---|---|---|
| `lcd(0x3F, ...)` | `0x3F` | I2C address of the LCD. Many modules use `0x27` instead. |
| `DIST_M` | `0.15` | Gap between the two sensors in meters (15 cm). |
| `SCALE` | `10.0` | Speed multiplier. `1` shows real speed. |
| `NORMAL_MAX` | `50.0` | Below this speed is "Normal Speed". |
| `OVERSPEED` | `60.0` | Above this speed is "OVERSPEEDING!". |
| `NO_CAR_MS` | `5000` | Idle time before "No car detected". |
| `NORMAL_SHOW` | `3000` | Display time for green and yellow results. |
| `OVERSPEED_SHOW` | `4000` | Display time for the red and buzzer result. |

## Code

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x3F, 16, 2);

const int S1 = 2, S2 = 3;
const int BUZZ = 8, GREEN = 9, RED = 10, YELLOW = 11;

const float DIST_M     = 0.15;   // measured gap between sensors (meters)
const float SCALE      = 10.0;   // speed multiplier (1 = real speed)
const float NORMAL_MAX = 50.0;   // below this = normal
const float OVERSPEED  = 60.0;   // above this = overspeeding

const unsigned long NO_CAR_MS      = 5000;  // idle time before "No car detected"
const unsigned long NORMAL_SHOW    = 3000;  // green / yellow display time
const unsigned long OVERSPEED_SHOW = 4000;  // red + buzzer display time

bool showing = false;
bool noCarShown = false;
unsigned long showStart = 0;
unsigned long showDuration = 0;
unsigned long lastActivity = 0;
float carSpeed = 0;

void allOff() {
  digitalWrite(GREEN, LOW);
  digitalWrite(RED, LOW);
  digitalWrite(YELLOW, LOW);
  noTone(BUZZ);
}

void showWaiting() {
  allOff();
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("Speed Detector");
  lcd.setCursor(0, 1); lcd.print("Waiting for car");
  noCarShown = false;
}

void showNoCar() {
  allOff();
  lcd.clear();
  lcd.setCursor(0, 0); lcd.print("No car detected");
  noCarShown = true;
}

// returns scaled speed in km/h, or -1 if measurement failed
float measureSpeed() {
  unsigned long t1 = micros();
  unsigned long timeout = millis() + 3000;
  while (digitalRead(S2) == HIGH) {
    if (millis() > timeout) return -1;
  }
  unsigned long t2 = micros();
  float seconds = (t2 - t1) / 1000000.0;
  if (seconds <= 0) return -1;
  float realKmph = (DIST_M / seconds) * 3.6;
  Serial.print("Real: "); Serial.print(realKmph, 1);
  Serial.print(" km/h  Scaled: "); Serial.println(realKmph * SCALE, 1);
  return realKmph * SCALE;
}

void showResult(float s) {
  allOff();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Speed: "); lcd.print((int)s); lcd.print(" km/h");
  lcd.setCursor(0, 1);

  if (s < NORMAL_MAX) {
    lcd.print("Normal Speed");
    digitalWrite(GREEN, HIGH);
    showDuration = NORMAL_SHOW;            // 3 sec
  } else if (s <= OVERSPEED) {
    lcd.print("Caution");
    digitalWrite(YELLOW, HIGH);
    showDuration = NORMAL_SHOW;            // 3 sec
  } else {
    lcd.print("OVERSPEEDING!");
    showDuration = OVERSPEED_SHOW;         // 4 sec (red + buzzer handled in loop)
  }
}

void setup() {
  Serial.begin(9600);
  pinMode(S1, INPUT); pinMode(S2, INPUT);
  pinMode(BUZZ, OUTPUT);
  pinMode(GREEN, OUTPUT); pinMode(RED, OUTPUT); pinMode(YELLOW, OUTPUT);
  lcd.init(); lcd.backlight();
  showWaiting();
  lastActivity = millis();
}

void loop() {
  unsigned long now = millis();

  if (!showing) {
    if (digitalRead(S1) == LOW) {                  // car reaches gate 1
      float s = measureSpeed();
      if (s > 0) {
        carSpeed = s;
        showResult(s);
        showStart = millis();
        showing = true;
      }
    } else if (!noCarShown && now - lastActivity >= NO_CAR_MS) {
      showNoCar();                                  // 5 sec with no car
    }
  } else {
    unsigned long elapsed = now - showStart;

    if (carSpeed > OVERSPEED) {                     // red flashing + buzzer
      bool on = ((elapsed / 250) % 2 == 0);
      digitalWrite(RED, on ? HIGH : LOW);
      if (on) tone(BUZZ, 1500); else noTone(BUZZ);
    }

    if (elapsed >= showDuration) {                  // time is up
      showing = false;
      lastActivity = millis();
      showWaiting();
    }
  }
}
```

## Hardware test sketch

Upload this first to check the LCD, LEDs, buzzer, and both sensors before running the main code.

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x3F, 16, 2);   // change to 0x27 if the screen stays blank

const int S1 = 2, S2 = 3;
const int BUZZ = 8, GREEN = 9, RED = 10, YELLOW = 11;

void setup() {
  pinMode(S1, INPUT); pinMode(S2, INPUT);
  pinMode(BUZZ, OUTPUT);
  pinMode(GREEN, OUTPUT); pinMode(RED, OUTPUT); pinMode(YELLOW, OUTPUT);

  lcd.init(); lcd.backlight();
  lcd.setCursor(0, 0); lcd.print("Test running");

  digitalWrite(GREEN, HIGH);  delay(600); digitalWrite(GREEN, LOW);
  digitalWrite(YELLOW, HIGH); delay(600); digitalWrite(YELLOW, LOW);
  digitalWrite(RED, HIGH);    delay(600); digitalWrite(RED, LOW);
  tone(BUZZ, 1500, 500);      delay(700);
}

void loop() {
  lcd.setCursor(0, 1);
  lcd.print("S1:");
  lcd.print(digitalRead(S1) == LOW ? "BLOCK " : "clear ");
  lcd.print("S2:");
  lcd.print(digitalRead(S2) == LOW ? "BLOCK" : "clear");
  delay(100);
}
```

Expected result: the LCD shows "Test running", the three LEDs light one after another, the buzzer beeps, and the second LCD line changes from `clear` to `BLOCK` when you put your hand in front of each sensor.

## I2C address scanner

Use this if the LCD backlight is on but no text appears. Open the Serial Monitor at 9600 baud and use the printed address in the `LiquidCrystal_I2C lcd(...)` line.

```cpp
#include <Wire.h>

void setup() {
  Serial.begin(9600);
  Wire.begin();
  Serial.println("Scanning...");
  for (byte a = 1; a < 127; a++) {
    Wire.beginTransmission(a);
    if (Wire.endTransmission() == 0) {
      Serial.print("Found device at 0x");
      Serial.println(a, HEX);
    }
  }
  Serial.println("Done");
}

void loop() {}
```

## Calibration

1. Set `SCALE = 1` and run the car through the gates a few times. Note the real speeds in the Serial Monitor (9600 baud).
2. Measure the real gap between the sensors and set `DIST_M` to match. To double-check, compare a run against a stopwatch: speed = gap / time between the gates.
3. Set `SCALE` to your car's model scale (for example 64 for 1:64, 32 for 1:32), or choose a value that makes a normal push land in the green range and a fast push land above 60.
4. Run slow, medium, and fast passes and confirm you get green, yellow, and red. If everything is red, lower `SCALE`. If everything is green, raise it.

A quick starting formula: `SCALE = 40 / (normal real speed in km/h)`.

## Troubleshooting

| Problem | Fix |
|---|---|
| LCD backlight on, no text | Turn the contrast screw on the back of the I2C board, then run the I2C scanner and use the address it prints (`0x3F` or `0x27`). |
| LCD blank and scanner finds nothing | Check that SDA goes to A4 and SCL to A5, and that the jumpers are firmly seated. |
| A sensor always says BLOCK | Turn the sensitivity screw on the IR module anticlockwise until its red LED turns off with nothing in front. Point it at open space away from walls and the table. |
| A sensor never detects the car | Move it closer (2 to 5 cm), add white tape to the car, or turn the sensitivity screw the other way. |
| Speeds look wrong | Re-measure `DIST_M` from sensor centre to sensor centre, and check that the car reaches IR 1 before IR 2. |
| LED does not light | Check the LED direction (long leg toward the resistor and pin) and that the resistor is connected. |
| No buzzer sound | Check that + goes to D8 and - goes to GND. |
| Upload fails | Select Board: Arduino Nano, and try Processor: ATmega328P (Old Bootloader). |
| Sensors trigger each other | Place a small cardboard divider between the two sensors, or angle them apart. |

## Ideas for upgrades

- Use laser modules and LDR modules instead of IR sensors for faster and more reliable detection
- Show a top-speed record or a car counter on the LCD
- Put the LCD in a cardboard speed-camera tower or traffic-cop robot
- Add a "Real / Scale" mode button
- Send readings to a phone with an ESP32

## Safety and honesty notes

- Use this as a model or educational demo. It is not a certified speed measurement device.
- When using `SCALE`, always label the reading as "scaled speed".
- If you test with real vehicles, stay safely beside the road and never place equipment on it.
