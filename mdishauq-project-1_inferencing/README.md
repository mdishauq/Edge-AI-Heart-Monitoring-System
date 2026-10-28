# Edge AI Heart Monitoring System

An embedded machine-learning inference project for classifying heart-monitoring sensor data on Arduino-compatible devices.

This repository contains an Edge Impulse model exported as an Arduino library. The model processes two-channel optical sensor data (`red` and `ir`) locally on a microcontroller and classifies the current signal or activity state as:

- `ACTIVE`
- `BACKGROUND`
- `RESTING`

> **Important:** This project is intended for educational and experimental purposes. It is not a medical device and must not be used for diagnosis, treatment, or clinical decision-making.

## Features

- On-device machine-learning inference
- Edge Impulse generated C++ inference SDK
- Arduino library format
- Two-channel optical sensor input:
  - Red optical signal
  - Infrared optical signal
- Quantized TensorFlow Lite model
- Spectral feature extraction using FFT
- Classification of three signal/activity states
- Support for resource-constrained embedded devices

## Model Details

| Property | Value |
|---|---|
| Input sensor | `red + ir` |
| Sensor type | Sensor fusion |
| Sampling frequency | 25 Hz |
| Sampling interval | 40 ms |
| Samples per channel | 25 |
| Total raw input values | 50 |
| Approximate window duration | 1 second |
| DSP processing | Spectral analysis with FFT |
| FFT length | 256 |
| Model type | Quantized TensorFlow Lite |
| Output classes | 3 |
| Classification threshold | 0.6 |

The model expects interleaved sensor data in this format:

```text
red_0, ir_0, red_1, ir_1, ..., red_24, ir_24
```

## Project Structure

```text
mdishauq-project-1_inferencing/
├── examples/
│   ├── esp32/
│   ├── nano_ble33_sense/
│   ├── nicla_sense/
│   ├── nicla_vision/
│   ├── portenta_h7/
│   ├── rp2040/
│   ├── sony_spresense/
│   └── static_buffer/
├── src/
│   ├── mdishauq-project-1_inferencing.h
│   ├── model-parameters/
│   │   ├── model_metadata.h
│   │   └── model_variables.h
│   ├── tflite-model/
│   │   ├── tflite_learn_887199_8_compiled.cpp
│   │   └── tflite_learn_887199_8_compiled.h
│   └── edge-impulse-sdk/
└── library.properties
```

## Requirements

- Arduino IDE or Arduino CLI
- A compatible Arduino board
- A two-channel optical sensor that provides red and infrared readings
- The sensor library required by the selected hardware
- Enough RAM and flash memory for the Edge Impulse SDK and model

The generated inference library does not define a specific optical sensor. Sensor acquisition code must be adapted to the hardware being used.

## Installation

### Arduino IDE

1. Download or clone this repository.
2. Open Arduino IDE.
3. Select **Sketch > Include Library > Add .ZIP Library...** if installing from a ZIP archive.
4. Select the target board under **Tools > Board**.
5. Install the required sensor-specific libraries.
6. Open an example sketch or create a new inference sketch.
7. Compile and upload the sketch.

Include the generated inference library in your sketch:

```cpp
#include <mdishauq-project-1_inferencing.h>
```

## Basic Inference Example

The classifier requires 50 floating-point values: 25 red-channel samples and 25 infrared-channel samples.

```cpp
#include <mdishauq-project-1_inferencing.h>

static float features[EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE];

int raw_feature_get_data(size_t offset, size_t length, float *out_ptr) {
    memcpy(out_ptr, features + offset, length * sizeof(float));
    return 0;
}

void setup() {
    Serial.begin(115200);

    while (!Serial) {
        delay(10);
    }

    Serial.println("Edge AI Heart Monitoring Inference");
}

void loop() {
    // Fill the array with sensor readings in this order:
    //
    // features[0] = red_0;
    // features[1] = ir_0;
    // features[2] = red_1;
    // features[3] = ir_1;
    // ...
    // features[48] = red_24;
    // features[49] = ir_24;

    signal_t signal;
    signal.total_length = EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE;
    signal.get_data = raw_feature_get_data;

    ei_impulse_result_t result = { 0 };

    EI_IMPULSE_ERROR error = run_classifier(
        &signal,
        &result,
        false
    );

    if (error != EI_IMPULSE_OK) {
        Serial.print("Inference failed with error: ");
        Serial.println(error);
        delay(1000);
        return;
    }

    Serial.println("Predictions:");

    for (size_t i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
        Serial.print("  ");
        Serial.print(result.classification[i].label);
        Serial.print(": ");
        Serial.println(result.classification[i].value, 5);
    }

    Serial.print("DSP time: ");
    Serial.print(result.timing.dsp);
    Serial.println(" ms");

    Serial.print("Classification time: ");
    Serial.print(result.timing.classification);
    Serial.println(" ms");

    delay(1000);
}
```

## Reading Predictions

Each prediction contains a label and a confidence value between `0.0` and `1.0`.

Example:

```text
Predictions:
  ACTIVE: 0.01234
  BACKGROUND: 0.08125
  RESTING: 0.90641
```

The label with the highest confidence is the model's predicted class. Applications should apply an appropriate confidence check before changing device behavior.

```cpp
if (result.classification[2].value >= 0.6f) {
    Serial.println("RESTING state detected");
}
```

Do not treat a prediction as reliable when the sensor is disconnected, poorly positioned, saturated, or producing noisy data.

## Sensor Data Requirements

Before running inference:

1. Sample both red and infrared channels at approximately 25 Hz.
2. Collect 25 samples from each channel.
3. Keep the two channels synchronized.
4. Store readings in interleaved order.
5. Avoid missing or duplicated samples.
6. Use the same sensor scaling and preprocessing used during model training.

The model was generated for the axes:

```text
red + ir
```

Changing the channel order, sampling rate, sensor type, or preprocessing can significantly reduce accuracy.

## Included Examples

The repository includes examples for several Arduino-compatible platforms:

- ESP32
- Arduino Nano 33 BLE Sense
- Arduino Nicla Sense
- Arduino Nicla Vision
- Arduino Portenta H7
- Raspberry Pi Pico/RP2040
- Sony Spresense
- Static-buffer inference

The static-buffer example is located at:

```text
examples/static_buffer/static_buffer/static_buffer.ino
```

It can be used to test the model with fixed feature data before connecting a physical sensor.

## Model Outputs

The current model provides three classification outputs:

```text
ACTIVE
BACKGROUND
RESTING
```

The model does not directly output:

- Heart rate in BPM
- Blood oxygen saturation (SpO2)
- ECG waveform data
- Medical alerts
- Clinical diagnoses

Those capabilities would require additional signal-processing logic or separately trained models.

## Troubleshooting

### Incorrect input size

The model requires:

```cpp
EI_CLASSIFIER_DSP_INPUT_FRAME_SIZE
```

For the current model, this is:

```text
25 samples x 2 channels = 50 values
```

### Low-confidence predictions

Check:

- Sensor placement
- Ambient light interference
- Sensor contact with the skin
- Red and infrared channel order
- Sampling frequency
- Missing or duplicated samples
- Sensor saturation
- Differences between training and deployment hardware

### Inference errors

Verify that:

- The generated header is included.
- The selected board has enough memory.
- The correct library folder is installed.
- Required sensor dependencies are installed.
- The signal length is exactly 50 values.
- The signal callback returns valid floating-point data.

## Edge Impulse Project

The generated model metadata identifies the Edge Impulse project as:

```text
Project owner: mdishauq
Project name: mdishauq-project-1
Project ID: 887199
Deployment version: 2
```

The generated inference implementation is located in:

```text
src/mdishauq-project-1_inferencing.h
src/model-parameters/
src/tflite-model/
src/edge-impulse-sdk/
```

## License and Usage

This repository contains software generated by Edge Impulse together with third-party SDK components.

The generated model metadata states that the model files are subject to the applicable Edge Impulse Terms of Service and may require an active Edge Impulse subscription. Review the applicable terms before redistributing or using the generated model:

- [Edge Impulse Terms of Service](https://edgeimpulse.com/legal/terms-of-service)
- [Edge Impulse Enterprise Terms of Service](https://edgeimpulse.com/legal/enterprise-terms-of-service)

The embedded Edge Impulse SDK and third-party components may have separate licenses. Refer to the license files inside:

```text
src/edge-impulse-sdk/
```

## Disclaimer

This project is provided for research, learning, and prototyping purposes only. It has not been validated as a medical device. Its predictions may be inaccurate and must not be used for medical diagnosis, monitoring, treatment, or emergency decisions.
