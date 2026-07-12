# 🤖 Autonomous Guided Vehicle (AGV) for Intelligent Warehouse Automation

> An intelligent warehouse robot capable of autonomously transporting goods using computer vision, path planning, real-time navigation, cloud connectivity, and embedded systems.

---

# 📌 Project Overview

Modern warehouses require intelligent systems that can transport inventory efficiently without constant human intervention. This project aims to build an **Autonomous Guided Vehicle (AGV)** capable of navigating a warehouse, identifying delivery destinations through QR codes, planning the shortest route, avoiding obstacles, delivering packages, and returning to its home station—all while maintaining real-time communication with a cloud backend.

Unlike conventional line-following robots, this AGV combines multiple robotics concepts including **embedded systems, graph algorithms, computer vision, cloud integration, and autonomous decision-making** into a single modular architecture.

---

# 🎯 Project Objectives

The primary objective of this project is to develop a modular and scalable warehouse robot capable of:

- Autonomous navigation inside a warehouse.
- Dynamic destination identification using QR Codes.
- Intelligent shortest-path computation.
- Accurate line following using PID control.
- Junction-based navigation.
- Real-time obstacle detection.
- Autonomous delivery mechanism.
- Cloud connectivity using Firebase.
- Returning safely to its charging/home station after every delivery.

---

# 🏗 System Architecture

The AGV consists of two independent embedded systems working together.

## 1️⃣ Main Controller (ESP32 Development Board)

The ESP32 Development Board acts as the robot's brain.

It is responsible for:

- Reading IR sensors.
- Following the warehouse path.
- Executing PID corrections.
- Detecting junctions.
- Computing navigation decisions.
- Controlling motors.
- Monitoring obstacles.
- Operating the delivery mechanism.
- Communicating with Firebase.
- Receiving destination information from the ESP32-CAM.
- Managing the overall robot state.

---

## 2️⃣ ESP32-CAM Module

The ESP32-CAM functions as the robot's vision system.

Its responsibilities include:

- Capturing camera frames.
- Detecting QR Codes.
- Decoding destination information.
- Sending the destination to the Main Controller through UART.

This separation keeps image processing independent from navigation, allowing both controllers to operate efficiently.

---

# 🔄 Complete Working Flow

The complete execution cycle of the AGV follows the sequence below.

```
Power ON
      │
      ▼
Initialize Hardware
      │
      ▼
Connect to Wi-Fi & Firebase
      │
      ▼
Wait for Delivery Request
      │
      ▼
Scan QR Code
      │
      ▼
Extract Destination
      │
      ▼
Generate Warehouse Graph
      │
      ▼
Compute Shortest Path (A*)
      │
      ▼
Begin Navigation
      │
      ▼
Line Following + PID
      │
      ▼
Detect Junction
      │
      ▼
Choose Correct Turn
      │
      ▼
Obstacle?
      │
 ┌────┴─────┐
 │          │
Yes        No
 │          │
Wait      Continue
 │          │
 └────┬─────┘
      ▼
Reach Destination
      │
      ▼
Open Delivery Tray
      │
      ▼
Confirm Delivery
      │
      ▼
Generate Return Path
      │
      ▼
Navigate Home
      │
      ▼
Idle
```

---

# 🧩 Software Architecture

Every software module has an independent responsibility.

## 🚗 Differential Drive

Controls the speed and direction of the left and right motor pairs, enabling forward motion, turning, and precise steering.

---

## ⚙ Motor Driver

Provides the low-level interface between the ESP32 and the TB6612FNG motor driver.

Responsibilities:

- PWM generation
- Direction control
- Speed control

---

## 📍 IR Sensor Processing

Reads the 8-channel IR sensor array and converts raw sensor readings into useful navigation information.

Responsibilities:

- Line detection
- Position estimation
- Sensor normalization

---

## 🎯 PID Controller

Continuously calculates steering corrections to ensure the robot remains centered on the line.

Responsibilities:

- Error calculation
- Proportional correction
- Integral compensation
- Derivative stabilization

---

## 🚦 Junction Detection

Identifies warehouse intersections using the IR sensor patterns.

Supported junctions include:

- Left Junction
- Right Junction
- Cross Junction
- Dead End
- End Marker

---

## 🗺 Warehouse Graph

Represents the complete warehouse as a graph consisting of nodes and edges.

Each node represents a junction.

Each edge represents a valid travel path.

---

## ⭐ A* Path Planner

Computes the shortest possible route between the current location and the destination.

Responsibilities:

- Graph search
- Cost calculation
- Heuristic evaluation
- Optimal path generation

---

## 🧭 Navigation Controller

Coordinates all navigation modules.

Responsibilities:

- Follow computed path
- Execute turns
- Advance between nodes
- Track robot position
- Handle navigation events

---

## 🚧 Obstacle Detection

Uses the ultrasonic sensor to detect objects blocking the robot.

Responsibilities:

- Stop movement
- Wait until path clears
- Resume navigation safely

---

## 📦 Delivery Manager

Controls the servo-operated delivery tray.

Responsibilities:

- Open tray
- Wait for package collection
- Close tray
- Confirm completion

---

## 📷 QR Detection

Uses the ESP32-CAM to identify destination QR Codes.

Responsibilities:

- Capture images
- Decode QR
- Validate destination
- Send destination via UART

---

## 🔄 UART Communication

Provides reliable communication between the ESP32-CAM and the Main Controller.

Responsibilities:

- Data transmission
- Message parsing
- Error handling

---

## ☁ Firebase Integration

Maintains communication with the cloud backend.

Responsibilities:

- Robot status updates
- Delivery requests
- Live monitoring
- Notifications

---

## 🏠 Return Home

After every successful delivery, the robot automatically computes the shortest route back to its home station.

---

# 🔩 Hardware Components

- ESP32 Development Board
- ESP32-CAM
- TB6612FNG Motor Driver
- 4 BO Motors
- 8-Channel IR Sensor Array
- HC-SR04 Ultrasonic Sensor
- SG90 Servo Motor
- LM2596 Buck Converter
- 18650 Battery Pack
- Robot Chassis

---

# 💻 Technologies Used

- Python
- C++
- ESP32
- OpenCV
- Firebase
- Git
- GitHub

---

# 🚀 Current Development Status

✅ Software architecture designed

✅ Core navigation modules implemented

✅ Line Following

✅ PID Controller

✅ Junction Detection

✅ A* Path Planning

✅ QR Detection

✅ UART Communication

✅ Firebase Integration

🚧 Hardware integration in progress

🚧 Repository restructuring in progress

🚧 Firmware modularization in progress

---

# 📖 Repository Note

The repository currently contains the generalized software modules and prototype implementations developed during the software design phase.

The folder structure is being migrated to the final modular architecture, therefore some directories may currently act as placeholders for future implementation.
