---
title: "How I run ROS 2 Humble on macOS with OrbStack"
date: 2026-10-01T11:27:00+08:00
draft: false
tags:
  - ROS 2
  - Docker
  - macOS
description: ""
---

Hi, This is Chin-Wei Kuan.

I am an aspiring computer science learner focused on **AI/Robotics**.

Documenting my technical growth here is a way to help myself to become a real profession.
Fake it till I make it. Hope this article also helps you. 

Also, If there is anything wrong, please feel free to correct me.
Cheer!

## Why containerize

Think about a scenario--you want to start a simulation project on Linux but your computer is Mac. What choices do you have?

First, you can simply install duo-OS, that is, divide half of hardware resource. It might be a little be wasting right?
Fortunately, there is option two -- Containerizing.

Containerizing is mature technique for developer who's os doesn't fit with the develope environment. Let's say. If your computer is like a house, Containerizing is seperating a independant room inside your house, with different operating environment and package as well.

In this case, I choose **OrbStack** to implement it for it is simpler on MacOS.

## Mental model

Les's begin with three things. I mix them up all the time at first.

**1.Image**
is like a frozen disk snapshot. It has Ubuntu, ROS 2 Humble, Gazebo, Python packages—everything the `Dockerfile` installs when you run `docker compose build`. You do not edit an image by hand; you rebuild it when the recipe changes.

**2.Container** 
is one running copy of that image. When you `./run_docker.sh`, Docker starts a fresh container. When you exit and the container is removed (`--rm`), anything you built *only inside* that container—like `colcon install`—is gone unless you build again.

**3.Bind mount** 
is the bridge between your Mac and the container. Our repo on the host is mounted at `/root/offroad_ws/src/ros2_offroad_follower`. Edit files in Cursor on macOS; the container sees the same files instantly. No image rebuild for code changes.


**`colcon build`** is separate from Docker build. It compiles *our* ROS packages (`offroad_bringup`, `offroad_perception`, …) into `install/` under `/root/offroad_ws`. That folder usually lives in the container filesystem, not in your git repo. New container requires to run `colcon build` again.


## Repository layout

All dev-environment wiring lives in `docker/`:
| File | Role |
|------|------|
| `Dockerfile` | Recipe for the **image**: apt ROS packages, Gazebo Fortress, pip (YOLO, torch), NumPy pin for `cv_bridge`. |
| `compose.yaml` | Recipe for **running** a container: amd64 platform, volumes, graphics env vars, privileged mode. Read by `docker compose`, not by you manually. |
| `run_docker.sh` | Mac entrypoint: sets `DISPLAY`, runs `docker compose run --rm offroad_dev`. |
| `setup_ml_deps.sh` | Safety check on shell login: re-pin NumPy/OpenCV if something broke compatibility with ROS `cv_bridge`. |


You almost never "open" `compose.yaml` day to day—you run the script, and Compose reads it for you.

## macOS workflow

This is the loop that actually worked for our simulation + perception stack—not two separate `./run_docker.sh` sessions.

**Terminal 1:**
```bash
cd path/to/ros2_offroad_follower/docker
./run_docker.sh
```
Inside the container:

```bash
cd /root/offroad_ws
colcon build --symlink-install
source install/setup.bash
ros2 launch offroad_bringup sim.launch.py
```

Leave this running.

**Terminal 2:** do not run `./run_docker.sh` again. Attach to the same container:
```bash
docker ps   # find the container running sim
docker exec -it <container_name_or_id> bash
source /root/offroad_ws/install/setup.bash
ros2 launch offroad_bringup perception.launch.py
```

On Mac, a second `./run_docker.sh` often means a second isolated ROS graph—you will not see each other's topics. One container, docker exec for extra shells—that is the pattern.

## Pitfalls

### Second `run_docker.sh`

**Symptom:** `ros2 topic list` looks empty across terminals, or nodes cannot talk.

**Fix:** Keep one long-lived container for sim, and use `docker exec` for other nodes. 

Note: This way looks not smart so far. I might figure out other way after finishing next milestone.

### NumPy 2 vs `cv_bridge`

ROS Humble's `cv_bridge` expects NumPy 1.x; pip ML stacks may pull NumPy 2.x.

**Symptom:** `AttributeError: _ARRAY_API not found` when launching the human tracker.

**Fix:** Pin in the `Dockerfile` (`numpy<2`, `opencv-python<5`) and run `docker compose build`. `setup_ml_deps.sh` on bash login is a backup.

### Missing `install/setup.bash`

A new container has no prior `colcon build`.

**Symptom:** `package 'offroad_perception' not found`.

**Fix:** `colcon build`, then `source install/setup.bash` in that container.

## What's next
With the shell in place, the next post walks the simulation layer: Gazebo trail world, Go2 URDF, ros_gz_bridge, and getting camera topics into ROS 2. After that we stack perception—YOLO, depth, and /target_human/state—but the container workflow above stays the same.

Cheer, and see you in the next one.