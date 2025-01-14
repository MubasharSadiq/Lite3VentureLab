# Lite3VentureLab

Code:
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# Copyright (c) 2023, Bettering Our Worlds (BOW) Ltd.
# All Rights Reserved
# Author: Daniel Camilleri <daniel.camilleri@bow.ltd>

import time
import bow_client as bow
import bow_utils
import sys
import logging
import cv2
from pynput import keyboard

# Global variables
stopFlag = False
window_names = dict()
windows_created = False

# A set to keep track of the pressed keys
pressed_keys = set()

# Keyboard event handlers
def on_press(key):
    try:
        pressed_keys.add(key.char)
    except AttributeError:
        pass

def on_release(key):
    try:
        pressed_keys.discard(key.char)
    except AttributeError:
        pass
    if key == keyboard.Key.esc:
        # Stop listener when ESC is pressed
        return False

# Start keyboard listener in a separate thread
listener = keyboard.Listener(on_press=on_press, on_release=on_release)
listener.start()

# Function to display all images from the robot's vision modality
def show_all_images(images_list):
    global windows_created, window_names

    if not windows_created:
        for i in range(len(images_list)):
            window_name = f"RobotView{i} - {images_list[i].source}"
            window_names[images_list[i].source] = window_name
            cv2.namedWindow(window_name)
        windows_created = True

    for img_data in images_list:
        if img_data.new_data_flag:
            cv2.imshow(window_names[img_data.source], img_data.image)

# Initialize logging and connect to the robot
log = bow_utils.create_logger("Bow Tutorial 2", logging.INFO)
log.info(bow.version())

myrobot, error = bow.quick_connect(pylog=log, modalities=["vision", "motor"])
if not error.Success:
    log.error("Failed to connect to robot: %s", error)
    sys.exit()

try:
    while True:
        # Sense: Get vision data from the robot
        image_list, err = myrobot.get_modality("vision", True)
        if not err.Success or len(image_list) == 0:
            continue

        show_all_images(image_list)

        # Decide: Prepare motor commands based on keyboard input
        motorSample = bow_utils.MotorSample()
        if 'w' in pressed_keys:
            print("Moving forward")
            motorSample.Locomotion.TranslationalVelocity.X = 0.2
        if 's' in pressed_keys:
            print("Moving backward")
            motorSample.Locomotion.TranslationalVelocity.X = -0.2
        if 'd' in pressed_keys:
            print("Rotate right")
            motorSample.Locomotion.RotationalVelocity.Z = -1.0
        if 'a' in pressed_keys:
            print("Rotate left")
            motorSample.Locomotion.RotationalVelocity.Z = 1.0
        if 'e' in pressed_keys:
            print("Strafe right")
            motorSample.Locomotion.TranslationalVelocity.Y = -1.0
        if 'q' in pressed_keys:
            print("Strafe left")
            motorSample.Locomotion.TranslationalVelocity.Y = 1.0

        # Act: Send motor commands to the robot
        myrobot.set_modality("motor", motorSample)

        # Allow for OpenCV window operations and refresh rate control
        cv2.waitKey(1)

except (KeyboardInterrupt, SystemExit):
    log.info("Shutting down...")
    stopFlag = True

finally:
    # Cleanup resources before exiting
    cv2.destroyAllWindows()
    myrobot.disconnect()
    bow.close_client_interface()
