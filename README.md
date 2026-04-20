# Low Voltage Conductor Breakage Detector

## Overview

This project implements a cost-effective and intelligent solution for detecting breakages in Low Voltage AC Distribution Overhead conductors. It leverages an ESP32 microcontroller, current and voltage sensors, and an on-device TensorFlow Lite anomaly detection model.

The system continuously monitors the electrical parameters of a power line. An AI model, trained to recognize normal operating conditions, runs directly on the device (Edge AI). If a deviation—such as a conductor break—is detected, the system automatically isolates the power supply via a relay, activates a local buzzer alarm, and is designed to send an SMS alert to relevant personnel using a GSM module.

## Key Features

*   **Real-time Monitoring:** Continuous sensing of current and voltage from the power line.
*   **Edge AI Anomaly Detection:** A lightweight autoencoder model runs on the ESP32 to detect faults without requiring cloud connectivity.
*   **Automated Fault Response:** Automatically triggers a relay to cut power and activates a buzzer upon fault detection.
*   **GSM Alerting:** Incorporates a SIM800L module to send SMS notifications for immediate response from authorities.
*   **Cost-Effective:** Built with widely available and affordable electronic components.

## Hardware Components

*   **Microcontroller:** ESP32
*   **Current Sensor:** ACS712
*   **Voltage Sensor:** ZMPT101B
*   **Actuators:** 5V Relay Module, Buzzer
*   **Connectivity:** SIM800L GSM Module

## Software & Technologies

*   **IDE:** Arduino IDE
*   **AI/ML Framework:** TensorFlow, TensorFlow Lite Micro
*   **Programming Languages:** Python (for training), C++ (for ESP32)
*   **Libraries:** Pandas, Scikit-learn

## Methodology

The system operates in three main stages: model training, on-device inference, and fault response.

### 1. AI Model Training

An autoencoder neural network is trained using the `train_model.py` script.

1.  **Data Loading:** Normal operating current readings are loaded from a `current_reading.csv` file.
2.  **Preprocessing:** The data is scaled to a range of 0 to 1 using `MinMaxScaler` from Scikit-learn.
3.  **Model Architecture:** A simple autoencoder is constructed with TensorFlow/Keras. It consists of an encoder that compresses the input data into a smaller representation and a decoder that attempts to reconstruct the original data from this compressed form.
    *   Input Layer (1 neuron)
    *   Encoder Layer (8 neurons, ReLU activation)
    *   Bottleneck Layer (4 neurons, ReLU activation)
    *   Decoder Layer (8 neurons, ReLU activation)
    *   Output Layer (1 neuron, Sigmoid activation)
4.  **Training:** The model is trained on the scaled "normal" data. It learns to minimize the reconstruction error (Mean Absolute Error) between its input and output.
5.  **Threshold Calculation:** After training, a reconstruction error threshold is calculated (e.g., the 95th percentile of errors on the training data). This threshold distinguishes normal operation from anomalous events.
6.  **Conversion:** The trained model is converted to the TensorFlow Lite format (`anomaly_detector.tflite`).

### 2. Model Conversion for Microcontroller

The `convert_model.py` script takes the `anomaly_detector.tflite` file and converts its binary content into a C-style byte array. This array is saved in the `model_data.h` header file, allowing the model to be compiled directly into the ESP32's firmware.

### 3. On-Device Inference and Fault Response

The `sketch_sep17a.ino` firmware runs on the ESP32 and performs the real-time detection.

1.  **Setup:** The firmware initializes the GPIO pins for the sensors and actuators. It then loads the AI model from `g_model_data` (the array in `model_data.h`) into memory using the TensorFlow Lite Micro library.
2.  **Sensing Loop:** In a continuous loop, the ESP32 reads the analog values from the ACS712 current sensor and the voltage sensor.
3.  **Inference:** The raw current reading is scaled to a 0.0-1.0 range and fed into the AI model's input tensor. The interpreter runs inference, and the reconstruction error (the absolute difference between the scaled input and the model's output) is calculated.
4.  **Fault Detection:** A fault condition is triggered if either of these conditions is met:
    *   The calculated `reconstructionError` exceeds the predefined `ANOMALY_THRESHOLD`.
    *   The `actual_voltage` drops below the safe `VOLTAGE_THRESHOLD`.
5.  **Alerting:** Upon detecting a fault, the system:
    *   Sets the relay pin (`RELAY_PIN`) to `LOW`, cutting off the power supply.
    *   Activates the buzzer (`BUZZER_PIN`) for a few seconds to provide an audible local alert.
    *   Halts execution to prevent continuous alerts until the system is manually reset.

## Code Structure

*   `train_model.py`: Python script to train the autoencoder model on normal current data and save it as a `.tflite` file.
*   `convert_model.py`: A utility script that converts the `.tflite` model into the `model_data.h` C header file.
*   `model_data.h`: Header file containing the TensorFlow Lite model as a `const unsigned char` array.
*   `sketch_sep17a.ino`: The main Arduino C++ code for the ESP32. It integrates sensor reading, model inference, and actuator control.

## Installation and Setup

1.  **Hardware Assembly:** Connect the sensors (ACS712, ZMPT101B), actuators (Relay, Buzzer), and GSM module (SIM800L) to the appropriate GPIO pins on the ESP32 as defined in `sketch_sep17a.ino`.
2.  **Train Model:** If you have new training data, place it in `current_reading.csv` and run the `train_model.py` script to generate a new `anomaly_detector.tflite` model.
    ```bash
    pip install pandas scikit-learn tensorflow
    python train_model.py
    ```
3.  **Convert Model:** Run the `convert_model.py` script to update the C header file.
    ```bash
    python convert_model.py
    ```
4.  **Arduino IDE Setup:**
    *   Install the Arduino IDE and add the ESP32 board manager.
    *   Install the `TensorFlowLite_ESP32` library.
5.  **Upload Firmware:** Open `sketch_sep17a.ino` in the Arduino IDE, ensure `model_data.h` is in the same directory, and upload the sketch to your ESP32.

---
✍️ **Author:** Shaik Aqibur Rahman
