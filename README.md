# Pikachu Seeker - Autonomous Robot Navigation & Exploration

This project implements an **autonomous robot navigation system** designed to locate and approach a Pikachu target within a simulated environment. The system leverages **ROS 2** (Robot Operating System 2) for communication and control, **YOLO** (You Only Look Once) for real-time object detection, and operates entirely on **RGB-only sensor data** provided by the **Unity ProsTwin Simulator**. The navigation logic is tailored to three distinct virtual room types.

## Demo Video

[Pikachu Seeker Demo Video (Google Drive)](https://drive.google.com/file/d/1P8Q12GI16QvG5qBnd-x8SgZRH5N-Ua1c/view?usp=sharing)

-----

## Implementation Strategy: RGB-Only Navigation

The core constraint across all navigation challenges (**Random Door Room**, **Random Living Room**, and **Fixed Living Room**) is the reliance solely on **RGB camera input** for both localization and obstacle avoidance. The general strategy involves:

1.  **Scanning:** Using a pre-defined pattern to search for the Pikachu target.
2.  **Detection:** Utilizing YOLO to identify the Pikachu bounding box.
3.  **Approach & Alignment:** Aligning the robot towards the target based on the bounding box's pixel offset.
4.  **Obstacle Avoidance:** Employing dual-detection logic to ensure collision-free movement (detailed in the **Random Living Room** section).
5.  **Final Approach:** Switching to a timed forward movement when the target is extremely close to prevent loss of detection.

-----

## Navigation Logic by Room Type

The robot's navigation strategy is customized for the unique layout of each simulated room.

### 1\. Random Door Room

This environment consists of a room with five doors, where the Pikachu target is hidden behind one of them. The robot must systematically check each door.

#### Rotation and Calibration System

  * **Rotation Functions:** Four functions are implemented for precise turns:
      * Left 90° / Right 90°
      * Left 180° / Right 180°
  * **Horizon Line Detection:** The turning functions use **horizon line detection** to estimate the rotation angle.
  * **Horizon Line Calibration (Crucial for Accuracy):**
      * Initial assumption was that the detected horizon line would be the **wall-floor boundary**. However, the detection may also capture the **wall-ceiling boundary**.
      * **Calibration Solution:** The camera frame is split into **upper and lower halves**. The calibration logic is inverted for the upper half because the perspective effect causes lines in the upper half to tilt in the opposite direction compared to lines in the lower half, even if the robot's tilt is the same. This compensates for the visual distortion to ensure a truly straight path.

#### Door Traversal

  * **Strategy:** The robot employs a systematic, **left-to-right traversal** of the 4 doors.
  * **Tracking:** A simple **Bitmap** (or boolean array) is used to track the visited status of the doors (doors `0` `1` `2` `3`).
  * **Goal:** The robot moves from door to door, marking each as `visited`. Once the Pikachu is detected by YOLO, the door traversal stops, and the robot locks onto the target for the final approach.

### 2\. Random Living Room

This room is generally more open but contains various obstacles. The Pikachu's location is random.

#### Scanning and Approach

  * **Initial Scan:** The robot performs a single, **360° counterclockwise scan** upon entering the room.
  * **Lock-on:** If the Pikachu is detected during the scan, the robot immediately stops spinning and initiates a direct approach.

#### Obstacle Avoidance (Area-Based Stagnation)

  * This system is key for collision avoidance using only RGB data.
  * **Collision Detection Logic:** The bounding box area of the detected Pikachu is continuously monitored. If the **area change ratio** falls below a defined **threshold** (e.g., 5%) for a specific duration, it indicates the robot is stuck.
  * **Calculation:**
    ```
    area_change_ratio = (current_area - previous_area) / previous_area
    ```
  * **Avoidance Maneuver:** When a collision is detected, the robot executes a sequence:
    1.  **Reverse** (2.0 seconds)
    2.  **Turn** (2.0 seconds)
    3.  **Forward** (1.5 seconds)

### 3\. Fixed Living Room

This version is identical to the Random Living Room in structure, but the Pikachu's position is fixed.

#### Navigation Logic

  * The **same navigation model** used for the **Random Living Room** is applied. The initial 360° scan quickly locates the fixed target, and the robot proceeds with the standard approach and avoidance logic.

#### Final Stopping Condition

  * **Close-Range Detection Loss:** The robot's final stopping mechanism is triggered when the Pikachu's bounding box becomes so large that **YOLO detection is lost** (i.e., the object fills the entire camera frame).
  * **Final Push:** Once detection is lost, the robot is programmed to move **forward for an additional few seconds** (e.g., 3 seconds) before stopping, ensuring it reaches the target even when it's too close to be reliably detected.

-----

## Key Design Decisions (Summary)

| Challenge | Solution |
| :--- | :--- |
| Ambiguous horizon line detection | Split frame into upper/lower halves, apply **inverted calibration logic** to the upper half to correct for perspective tilt. |
| Systematic door exploration | Left-to-right traversal order tracked using a **simple bitmap**. |
| Collision detection without depth sensor | Monitor **Pikachu bounding box area growth rate**. Stagnation implies the robot is stuck. |
| Target loss when extremely close | **Ignore detection** and switch to a timed **final approach** (e.g., 3s) once the bounding box reaches a large size (e.g., 50,000 px²). |
| Ensuring robust obstacle avoidance | Dual-method (RGB image similarity + Area Stagnation) with **OR logic** to trigger avoidance. |

-----

## Project Structure

```
workspace/
├── pikachu_nav_door_room.py     # Navigation logic for the Random Door Room
├── pikachu_nav_living_room.py   # Navigation logic for Random/Fixed Living Rooms
├── yolo_detection_node.py       # ROS 2 wrapper for YOLO object detection
└── setup.py                     # ROS 2 package setup file
```

