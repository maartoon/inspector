# Human-in-the-Loop Assistive Navigation Framework for UAS-Based Infrastructure Visual Inspection

## Overview

Unmanned Aerial Systems (UAS) are standard tools for infrastructure inspection. However, fully autonomous flights are often restricted by regulatory constraints. This framework allows human pilots to maintain manual control of the drone while providing **Augmented Reality (AR) visual guidance** and **real-time 3D inspection quality monitoring**. This ensures complex pre-planned trajectories are executed accurately, resulting in high-quality data for Structure-from-Motion (SfM) 3D reconstruction.

This work is based on the research paper: *"A Human-in-the-Loop Assistive Navigation Framework for UAS-Based Infrastructure Visual Inspection"* by Martin Xu, Yuxiang Zhao, Zixin Wang, and Mohamad Alipour.

---

## Key Features

### 1. AR-Assisted Visual Navigation
The application projects 3D geographic coordinates onto the drone's 2D First-Person View (FPV) using real-time RTK telemetry and camera attitude matrices.
* **On-Screen Waypoints:** A dynamic, pulsing red beacon indicates the target viewpoint, scaling in size as the drone approaches. The distance indicator turns green when the target threshold (< 0.5m) is reached.
* **Off-Screen Guidance:** If the waypoint is outside the camera's Field of View (FOV), the system renders a dynamic directional arrow and a guiding lane at the edge of the screen to guide the pilot's yaw and pitch.
* **3D Synthetic Alignment:** Renders a virtual view of the 3D target structure aligned with the current viewpoint to help the pilot manually adjust the drone/gimbal orientation before capture.

### 2. Real-Time Mesh-Coupled Quality Monitoring
The application evaluates inspection progress *only* when the camera shutter is triggered, ensuring the heatmap accurately reflects captured data rather than simple camera panning.
* **View Redundancy Tracking:** Computes how many times a structural face has been captured successfully. It factors in distance constraints (< 50m), inclination angles (< 60°), camera FOV, and Möller-Trumbore occlusion checks. 
* **Triangulation Uncertainty:** Calculates theoretical geometric uncertainty using the Fisher Information Matrix (trace of the inverse H matrix) based on the spatial distribution of the captured viewpoints.

### 3. Edge-Computing Architecture
All spatial calculations, 3D-to-2D projections, and mesh updates occur locally on the remote controller utilizing Kotlin Coroutines. This eliminates the need for a continuous backend server connection, significantly reducing latency and improving field robustness.

---

## System Architecture

* **Sensor & Data Layer:** Ingests pre-computed trajectory data, drone RTK GNSS, gimbal attitude, and live camera feed via the DJI Mobile SDK (MSDK v5).
* **Application Layer:** * `LocationManager.kt`: Subscribes to MSDK calls, capturing state and shutter events, and transforming WGS-84 to local UTM coordinates using Proj4J.
  * `WaypointProjection.kt`: Handles the mathematical projection of World ENU coordinates into the drone's body frame and subsequently into the 2D camera pixel space.
  * `CoverageManager.kt`: Loads the structural OBJ file, calculates FOV visibility constraints, updates H-matrices for uncertainty, and groups the mesh into visual redundancy layers.
* **UI Layer:** Renders the AR overlays (`WaypointOverlayWidget.kt`) and 3D Sceneform meshes (`DynamicCoverageMesh.kt`, `DynamicUncertaintyMesh.kt`) without blocking the main thread.

---

## Hardware and Software Stack

**Hardware Supported:**
* DJI Mavic 3 Enterprise / Mavic 3T (with RTK module attached)
* DJI RC Plus 2 (or compatible Android-based smart controller)

**Software & Libraries:**
* **Language:** Kotlin
* **SDK:** DJI Mobile SDK (MSDK v5)
* **3D Rendering:** ARCore Sceneform (`SceneView`)
* **Geospatial Transform:** Proj4J (EPSG:4326 to EPSG:32616 UTM Zone 16N)
* **Concurrency:** Kotlin Coroutines (`Dispatchers.Main` and `Dispatchers.IO`)

---

## UI Color Mapping

The application provides live 3D feedback mapped directly over the target structure. 

**View Redundancy Map (`DynamicCoverageMesh.kt`):**
* `Level 0`: Uncovered (Transparent dark grey)
* `Level 1 - 2`: Insufficient (Saturated Pink -> Crimson)
* `Level 3 - 5`: Good (Vivid Ruby -> Dark Red)
* `Level 6+`: Highly Redundant (Deep Maroon)

**Triangulation Uncertainty Map (`DynamicUncertaintyMesh.kt`):**
* `High Uncertainty`: Light blue
* `Medium Uncertainty`: Sky blue
* `Very Low Uncertainty`: Navy blue

---

## Flight Test Dataset & EXIF Metadata

To validate the assistive navigation framework, on-site reconstruction missions were conducted on the façade of the Civil and Environmental Engineering Building at the University of Illinois Urbana-Champaign. 

The sample dataset containing the original photos captured during these flight tests, along with their embedded EXIF metadata (which the application utilizes for real-time coverage evaluation), is publicly available.

**[Flight Test Photos & EXIF Data (Google Drive)](https://drive.google.com/drive/folders/1mHXNIUlSdyawh08TLqmvsoyvGBiv1ltC?usp=share_link)**

## Telemetry & Pose Data (CSV)
In addition to the raw photos, the dataset includes localized spatial data extracted during the flight:
* **`trajectory_poses.csv`**: A continuous high-frequency log of the drone's flight path.
* **`capture_poses.csv`**: The recorded pose of the drone at the exact moment the camera shutter was triggered.

Both CSV files share the following column structure:
* `timestamp`: Unix timestamp (in milliseconds) of the recorded pose.
* `x, y, z`: The localized 3D spatial coordinates (in meters) converted from the global RTK GNSS data.
* `qx, qy, qz, qw`: The camera's attitude/orientation represented as a quaternion, derived from the drone's gimbal and magnetometer. 
---
