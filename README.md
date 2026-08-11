Engineering materials
====

This repository contains engineering materials of a self-driven vehicle's model participating in the WRO Future Engineers competition in the season 2026.

## Content

* `t-photos` contains 2 photos of the team (an official one and one funny photo with all team members)
* `v-photos` contains 6 photos of the vehicle (from every side, from top and bottom)
* `video` contains the video.md file with the link to a video where driving demonstration exists
* `schemes` contains one or several schematic diagrams in form of JPEG, PNG or PDF of the electromechanical components illustrating all the elements (electronic components and motors) used in the vehicle and how they connect to each other.
* `src` contains code of control software for all components which were programmed to participate in the competition
* `models` is for the files for models used by 3D printers, laser cutting machines and CNC machines to produce the vehicle elements. If there is nothing to add to this location, the directory can be removed.
* `other` is for other files which can be used to understand how to prepare the vehicle for the competition. It may include documentation how to connect to a SBC/SBM and upload files there, datasets, hardware specifications, communication protocols descriptions etc. If there is nothing to add to this location, the directory can be removed.

## Introduction

The vehicle is controlled by a Raspberry Pi 4B running Python 3. A LEGO Build HAT sits on the Pi's 40-pin header and drives all motors and sensors over a single serial link. Two independent programs implement the two competition rounds, plus one tool used only during development.

Modules
File	Role
src/open_challenge.py	Open Challenge — three laps between the walls
src/obstacle_challenge.py	Obstacle Challenge — passing the traffic signs
src/manual_drive.py	Keyboard-driven calibration and logging tool

Each program is a single file with the same internal shape: a setup phase, then one perception-then-state-machine loop per camera frame. Nothing is threaded; the loop runs at roughly 20 Hz and every decision is made from the readings of the current frame.

open_challenge.py holds the vehicle between the two walls using the difference of the two ultrasonic readings in a PD controller, detects each corner from the amount of black pixels in a forward region of interest, and counts laps from crossings of the orange or blue floor lines. It moves through four states: CALIBRATE (straighten and settle), FOLLOW (hold lateral position), TURN (full lock through the corner), and back to CALIBRATE.

obstacle_challenge.py tracks the traffic signs one at a time. Only the largest contour in the frame is ever tracked, because apparent area falls off as the inverse square of distance and the largest sign is therefore always the nearest one. Three states: SEARCH drives straight until a sign is large enough to accept, APPROACH steers so the sign sits on the image centre line, and ORBIT swings out for a fixed time on the side required by rule 9.19 — left for green, right for red — then counter-steers while looking for the next sign.

manual_drive.py never runs during a round. It drives the vehicle from the keyboard while displaying live contour areas with estimated distances, the commanded versus actual steering angle, and accumulated drive distance. It records the raw camera view to AVI and synchronised sensor readings to CSV, so thresholds can be tuned away from the competition field.

Relation to the electromechanical components
Software element	Hardware
steer_motor — steer_to()	SPIKE L angular motor, Build HAT port A, front Ackermann linkage
drive_motor — drive()	SPIKE L angular motor, Build HAT port D, rear axle through a gear train
left_sensor, right_sensor — read_distance()	Ultrasonic sensors, Build HAT ports B and C
cap — OpenCV VideoCapture	USB camera, 640×480 MJPG
SimpleButton	Push button on GPIO6 (physical pin 31), 3.3 V through 660 Ω, internal pull-down

Three couplings between software and hardware are worth stating explicitly, because each one caused a failure before it was understood:

The GPIO backend must be fixed before import buildhat. The buildhat library uses gpiozero to reset the HAT over GPIO4 at import time. If gpiozero selects RPi.GPIO — which it may do automatically — the import fails with Not running on a RPi!. Both programs therefore call init_pin_factory() at the top of the file, trying lgpio, pigpio, native and rpigpio in order and installing the first one that initialises.

The Build HAT reserves GPIO 0, 1, 4, 14, 15, 16 and 17. The start button uses GPIO6, which is outside that set. The button is edge-triggered in software — five consecutive samples reading released, then five reading pressed — because a floating pin otherwise registers as pressed the moment the program starts, and the vehicle would launch by itself.

Steering commands are rate-limited. steer_to() clamps both the angle and the change per frame. The rate limit protects the servo from sensor noise, but it also bounds how fast a manoeuvre can be executed: reversing lock to lock consumes several frames, which is a significant fraction of a 0.7 s avoidance swing.

Building and uploading the code

The code is Python and requires no compilation. Preparing a Raspberry Pi from a fresh Raspberry Pi OS image:

bash
sudo apt update
sudo apt install -y python3-opencv python3-gpiozero pigpio python3-pigpio
sudo systemctl enable --now pigpiod
pip3 install buildhat

Copy the contents of src/ to /home/pi on the Pi, over VNC, SSH or a USB drive. To run a round manually:

bash
python3 /home/pi/open_challenge.py        # or obstacle_challenge.py

The program initialises the camera and the Build HAT, then waits on a READY screen until the start button is pressed, as required by rule 9.11. Driving begins 0.5 s after the press.

During a competition round no keyboard or display may be connected (rules 11.6 and 11.10), so the program is installed as a systemd service that starts at power-on:

bash
sudo cp other/wro.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable wro

The unit sets WRO_HEADLESS=1, which suppresses all preview windows, and Restart=no, so the vehicle does not restart itself after three laps while the judges are still scoring. Change the ExecStart line to select which round to run. Allow about 40 seconds from power-on to the ready state; rule 12.10 gives 90 seconds of preparation, so the vehicle should be switched on before it is placed on the mat.
