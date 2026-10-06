> **About this repository.** This is an overview repository: it contains documentation and demo videos only. The project is being carried out jointly with Universidad Tecnológica Nacional, Facultad Regional San Francisco (UTN FRSFCO), and for the time being its development is private, so the source code is not published here.

# LSA on Unitree R1 — Vision-Driven Arm and Hand Retargeting

**Making a humanoid robot copy human arm and hand motion from video, as the first step towards reproducing Argentine Sign Language (LSA).**

This document summarizes what the project is, what I have built so far, the technical decisions behind it, and what comes next. It is written so that someone with no context can understand the state of the work in a few minutes.

---

## Overview

The goal is to reproduce Argentine Sign Language (LSA) hand configurations on a **Unitree R1 EDU** humanoid equipped with **BrainCo Revo2** dexterous hands.

The approach is deliberately **deterministic**: computer vision extracts body and hand landmarks from video, and geometric retargeting converts them into joint commands. There is no reinforcement learning or learned policy in the loop. Every stage can be inspected, measured and debugged on its own.

All work is validated in simulation first, so the pipeline can be developed without needing the physical robot on the bench and without risking the hardware.

---

## Demo

### 1. R1 arms following a person from video

https://github.com/user-attachments/assets/8ae4ac4b-db25-487c-8f31-930bc0e14fb1

The simulated R1 upper body reproduces the arm motion of a person in a recorded video. Arm landmarks are detected with MediaPipe, retargeted to the robot's proportions and solved with inverse kinematics.


### 2. Revo2 hand teleoperation

https://github.com/user-attachments/assets/3974f00b-83c0-4d12-b516-8355b9ddb00c

The simulated BrainCo Revo2 hand driven finger by finger from a gamepad through ROS 2, using the vendor's original joint architecture (6 commanded joints, 5 mimic joints).

---

## My Role

Design and implementation of the whole pipeline to date:

- Simulation bring-up of the Revo2 hand (right, left and dual) in Gazebo with `ros2_control`
- Gamepad teleoperation node for the hand, with ramped position commands
- Simulation of the R1 upper body in MuJoCo with `ros2_control`
- Inverse kinematics for the R1 arms
- Arm pose extraction from video and geometric retargeting to the robot
- Streaming joint rate limiter between retargeting output and the controllers
- Root-cause analysis of simulator issues, and written documentation of every decision so the work can be replicated

---

## System Architecture

```
[Reference video of a person]
            |
            v
[Vision]  MediaPipe Pose  ........ shoulder, elbow, wrist
          MediaPipe Hands ........ 21 hand landmarks        (in progress)
            |
            |  recorded landmarks (.npz)
            v
[Geometric retargeting]
   human segment directions applied to the robot's own link lengths
            |
            +--> ARMS
            |      IK (Pinocchio + SciPy)
            |        -> joint targets
            |        -> joint rate limiter
            |        -> ros2_control forward controllers
            |        -> MuJoCo (R1 upper body)
            |
            +--> HANDS
                   Revo2 joint angles            (today: gamepad teleop)
                     -> command ramp
                     -> ros2_control position controller
                     -> Gazebo Fortress (Revo2 hand)
```

---

## What Has Been Built

### 1. Revo2 hand simulation and teleoperation

- Revo2 hand running in Gazebo Fortress through `gz_ros2_control` with a `JointGroupPositionController`.
- A dedicated bring-up package owns its controller configuration and minimal xacro wrappers for right, left and dual-hand setups. The vendor description is kept unmodified at a pinned commit.
- A teleoperation node maps gamepad axes to the 6 active joints (two thumb joints plus index, middle, ring and pinky). The 5 distal joints follow through native mimic joints.
- All commands are applied as gradual ramps, never as instantaneous jumps (see Technical Challenges).

### 2. R1 upper-body simulation

- R1 upper body in MuJoCo, controlled through `ros2_control`, with waist and head locked to focus on the arms.
- A joint rate limiter node sits between target topics and the forward controllers, so targets can be streamed continuously while joint velocities stay bounded.

### 3. Arm inverse kinematics

- IK solved with Pinocchio for kinematics and SciPy as the optimizer, following the cost formulation Unitree uses in its teleoperation stack.
- IK and simulation share the same URDF, so what the solver assumes is exactly what the simulator runs.

### 4. Arm pose from video

- A standalone vision module extracts shoulder, elbow and wrist landmarks from recorded video with MediaPipe Pose.
- Retargeting uses the direction of each human arm segment combined with the robot's own measurements, instead of copying positions directly. This removes the dependency on the person's body proportions.
- A ROS 2 node replays the recorded landmarks and drives the simulated arms through IK.

---

## Technical Stack

| Layer | Technology |
|---|---|
| Robot platform | Unitree R1 EDU, BrainCo Revo2 dexterous hands |
| Middleware | ROS 2 Humble (Ubuntu 22.04) |
| Hand simulation | Gazebo Fortress, `gz_ros2_control` |
| Body simulation | MuJoCo, `mujoco_ros2_control` |
| Kinematics | Pinocchio, SciPy |
| Vision | MediaPipe (Pose, Hand Landmarker) |
| Languages | Python, URDF/xacro |

---

## Technical Challenges Solved

- **Joints freezing in simulation.** Hand joints would become permanently unresponsive until the simulation was restarted. I traced it to a known, unfixed Gazebo Fortress / DART issue ([gz-sim#1684](https://github.com/gazebosim/gz-sim/issues/1684)): a velocity-controlled joint that hits its physical limit stops responding. Large instantaneous position commands were triggering it. The fix was a mandatory ramp on every hand command.
- **A mimic-joint "bug" that was not a bug.** Distal joints appeared to jump to their limit on small commands. Instead of accepting a workaround, I kept investigating and showed it was an artifact of the test method (publishing the final value in a single step). With stepped commands the native mimic joints converge correctly, so the vendor's 6-joint architecture was kept and an 11-joint rewrite was discarded.
- **Keeping vendor code untouched.** Controller configuration and xacro wrappers were moved into my own bring-up package, leaving the vendor repository pinned and unmodified. This also fixed the left-hand configuration.
- **IK without breaking the ROS environment.** The reference implementation relies on CasADi and IPOPT installed through conda, which does not mix well with a ROS 2 Humble system install. I reimplemented the same cost formulation with pip-installed Pinocchio and SciPy.
- **One model for solver and simulator.** Using the project's own URDF for IK avoids subtle mismatches between the model the solver reasons about and the one being simulated.
- **Smooth motion from streamed targets.** Direct commands produced near-instant arm motion. A streaming velocity limiter was chosen over a trajectory controller because targets arrive continuously from vision rather than as planned trajectories.
- **Retargeting across different body proportions.** Using segment directions plus robot link lengths gives a reachable target regardless of the size of the person in the video.

---

## Status

**Active development.**

| Stage | State |
|---|---|
| Revo2 hand simulation + gamepad teleoperation | Done |
| R1 upper-body simulation + arm IK | Done |
| R1 arms copying a person from video | Working, with known limitations |
| Hand landmark detection (MediaPipe) | In progress |
| Revo2 hands mounted on the simulated R1 | Next |
| Finger retargeting from landmarks to Revo2 joints | Planned |
| Deployment on the physical R1, using its onboard camera | Planned |

Known limitations of the current arm tracking: depth is not corrected yet when working from a single-camera video, and the rate limiter speed still needs tuning.

Once the deterministic pipeline is complete, a separate project will tackle the same problem with imitation / reinforcement learning, to compare both approaches.

---

## Author

**Juan Cruz Armando** — Electronics Engineer & Embedded Software Developer

cruzjuanarmando@gmail.com · [github.com/JuanCruzz7](https://github.com/JuanCruzz7)
