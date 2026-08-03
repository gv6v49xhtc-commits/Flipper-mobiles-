/*
  gold_detector.ino
  Simple gold/metal detection prototyping sketch.
  - Reads analog sensor on A0
  - Applies a moving average smoothing
  - Triggers LED and buzzer when signal > threshold
  - Sends measurements over Serial for logging/plotting

  Wiring:
  - Sensor/coiled pickup -> analog pin A0 (with appropriate conditioning circuit)
  - LED -> digital 13 (with resistor)
  - Buzzer -> digital 12 (active buzzer or transistor driver)
*/

const int sensorPin = A0;
const int ledPin = 13;
const int buzzerPin = 12;

const int sampleWindow = 10;     // moving average window size
int samples[sampleWindow];
int sampleIndex = 0;
long sampleSum = 0;

int baseline = 0;                // baseline reading (set during calibration)
int threshold = 80;              // detection threshold above baseline — tune this

unsigned long lastCalTime = 0;
const unsigned long calInterval = 10000; // auto-recalibrate every 10s if needed

void setup() {
  pinMode(ledPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
  Serial.begin(115200);
  // initialize samples
  for (int i = 0; i < sampleWindow; ++i) {
    samples[i] = analogRead(sensorPin);
    sampleSum += samples[i];
  }
  baseline = sampleSum / sampleWindow;
  lastCalTime = millis();
  Serial.println("gold_detector ready");
  Serial.print("baseline,"); Serial.println(baseline);
}

void loop() {
  int val = analogRead(sensorPin);

  // update moving average
  sampleSum -= samples[sampleIndex];
  samples[sampleIndex] = val;
  sampleSum += samples[sampleIndex];
  sampleIndex = (sampleIndex + 1) % sampleWindow;
  int avg = sampleSum / sampleWindow;

  int delta = avg - baseline; // positive when signal rises
  bool detected = (delta > threshold);

  // outputs
  digitalWrite(ledPin, detected ? HIGH : LOW);
  if (detected) {
    tone(buzzerPin, 2000); // beep when detected
  } else {
    noTone(buzzerPin);
  }

  // send telemetry: timestamp,raw,avg,baseline,delta,detected
  unsigned long t = millis();
  Serial.print(t); Serial.print(',');
  Serial.print(val); Serial.print(',');
  Serial.print(avg); Serial.print(',');
  Serial.print(baseline); Serial.print(',');
  Serial.print(delta); Serial.print(',');
  Serial.println(detected ? 1 : 0);

  // periodic (optional) auto-baseline adjustment — comment out if undesired
  if (millis() - lastCalTime > calInterval) {
    // simple drift compensation: move baseline slowly toward current avg
    baseline = (baseline * 7 + avg) / 8;
    Serial.print("baseline,"); Serial.println(baseline);
    lastCalTime = millis();
  }

  delay(25); // sample rate ~40Hz; adjust as required
}


