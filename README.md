# UR3 Teleop + Dataset Recorder cho Pi0.5

Hệ thống điều khiển UR3 bằng HTC Vive tracker và thu data demonstration để train mô hình **Pi0.5** (Vision-Language-Action model của Physical Intelligence).

---

## 📋 Mục Lục

1. [Tổng Quan](#1-tổng-quan)
2. [Yêu Cầu Phần Cứng](#2-yêu-cầu-phần-cứng)
3. [Yêu Cầu Phần Mềm](#3-yêu-cầu-phần-mềm)
4. [Cấu Trúc Dự Án](#4-cấu-trúc-dự-án)
5. [Cài Đặt Lần Đầu](#5-cài-đặt-lần-đầu)
6. [Kiểm Tra Phần Cứng](#6-kiểm-tra-phần-cứng)
7. [Calibration (Hiệu Chuẩn)](#7-calibration-hiệu-chuẩn)
8. [Workflow Thu Data](#8-workflow-thu-data)
9. [Định Dạng Dataset](#9-định-dạng-dataset)
10. [Troubleshooting](#10-troubleshooting)
11. [Roadmap](#11-roadmap)

---

## 1. Tổng Quan

### Mục tiêu

Thu thập dữ liệu demonstration cho UR3 thực hiện task gắp vật, để fine-tune **Pi0.5** — một Vision-Language-Action model có thể học từ ít demonstration để robot tự thực hiện task.

### Cách hoạt động

```
HTC Vive Tracker (tay người cầm)
        ↓
    OpenVR → ROS2 TF
        ↓
    Vive pose → Robot target pose (qua calibration)
        ↓
    ur_rtde → UR3 di chuyển theo Vive
        ↓
    Record: 2 camera + joint state + TCP pose → Dataset
```

Người dùng cầm tracker di chuyển trong không khí → robot UR3 mirror chuyển động → 2 camera (front + wrist) ghi lại scene → dataset được lưu để train Pi0.5.

### Pipeline khi chạy

7 terminal chạy đồng thời, mỗi terminal 1 node ROS2:

| Terminal | Node | Vai trò |
|---|---|---|
| T1 | `vive_tf_and_joy_ros2.py` | Đọc Vive qua OpenVR → publish TF + button |
| T2 | `frame_as_posestamped_ros2.py` | Convert TF → PoseStamped |
| T3 | `vive_ur5_teleop_params.py` | Apply calibration → publish target pose |
| T4 | `ur_follow_using_class_ros2.py` | UR3 follow target qua ur_rtde |
| T5 | `realsense2_camera_node` | Camera front (top-down) |
| T6 | `usb_cam_node_exe` | Camera wrist (gripper) |
| T7 | `record_all.py` | GUI record + lưu dataset |

---

## 2. Yêu Cầu Phần Cứng

### Robot
- **Universal Robots UR3** với network IP `192.168.1.1`
- Polyscope đã enable External Control / RTDE

### VR Tracking (HTC Vive)
- **2 Lighthouses** (Base Stations) gắn trên tường, hướng vào không gian làm việc, cách nhau 2-5m
- **1 Vive Tracker 3.0** (hoặc Tracker 2.0) cầm tay
- **Headset** (chỉ dùng để init SteamVR, không cần đeo)

### Camera (BẮT BUỘC)
- **Intel RealSense D435I** — Front cam (top-down nhìn xuống bàn)
- **Logitech C922 Pro** (hoặc webcam USB tương tự) — Wrist cam (gắn lên gripper)

### Máy tính
- Ubuntu 22.04
- ⚠️ **USB 3.0 cho Realsense** — cáp USB-C → USB-A 3.0. Nếu dùng cáp USB 2.0 sẽ chỉ chạy 15fps thay vì 30fps
- ⚠️ **2 camera nên cắm vào 2 USB controller khác nhau** (check bằng `lsusb -t`) để không tranh bandwidth

### Setup phòng (gợi ý)
```
       Lighthouse 1                     Lighthouse 2
            \                               /
             \                             /
              \    [Workspace UR3]        /
               \   [Bàn làm việc]        /
                \                       /
                 \                     /
                  [Khu vực Vive tracker]
                  (người vận hành đứng ở đây)
```

---

## 3. Yêu Cầu Phần Mềm

### Hệ điều hành
- Ubuntu 22.04 LTS
- ROS2 Humble

### Python packages
```bash
pip install ur_rtde scipy numpy opencv-python openvr pynput
```

### ROS2 packages
```bash
sudo apt install ros-humble-realsense2-camera ros-humble-usb-cam \
                 ros-humble-tf-transformations ros-humble-cv-bridge
```

### SteamVR
- Cài qua Steam
- Phải mở SteamVR TRƯỚC khi chạy ROS node
- Trackers/lighthouses phải show xanh trong SteamVR status

### URBasic / ur_rtde
- `ur_rtde` đã được dùng thay cho URBasic (URBasic bị bug với realtime control)
```bash
pip install ur_rtde
```

---

## 4. Cấu Trúc Dự Án

```
~/ur5_teleop_vive/                          ← ROS workspace
├── install/                                ← colcon install (cần source)
├── src/
│   └── ur5_teleop_vive/
│       ├── msg/
│       │   └── Xyzrpy.msg                  ← Custom msg cho TCP pose
│       └── ...
└── ur5_teleop_vive/
    └── ur5_teleop_vive/
        └── thesis_code/                    ← Code chạy chính
            ├── vive_tf_and_joy_ros2.py
            ├── frame_as_posestamped_ros2.py
            ├── vive_ur5_teleop_params.py
            ├── ur_follow_using_class_ros2.py
            ├── record_all.py               ← Dataset recorder
            ├── camera_check.py             ← Test camera
            ├── calib_4x4.py                ← Hiệu chuẩn full 4x4
            ├── calib_manual.py             ← Hiệu chuẩn yaw đơn giản
            ├── world_alignment_matrix.txt  ← Ma trận calibration
            └── dataset/                    ← Output thu được
                └── <task_name>/
                    └── episodes/
                        └── ep_NNNN/
                            ├── images/
                            │   ├── front/
                            │   └── wrist/
                            ├── frames.jsonl
                            └── meta.json
```

### Topics ROS2 quan trọng

| Topic | Type | Publisher | Vai trò |
|---|---|---|---|
| `/right_controller_as_posestamped` | PoseStamped | frame_as_posestamped | Vive pose sau alignment |
| `/vive_right` | Joy | vive_tf_and_joy | Button trigger (Ctrl_R) |
| `/ur_target_pose` | PoseStamped | vive_ur5_teleop | Target gửi cho robot |
| `/ur_actual_pose` | Xyzrpy | ur_follow | TCP pose thực tế |
| `/ur_joint_states` | JointState | ur_follow | 6 joint positions |
| `/camera/camera/color/image_raw` | Image | realsense2_camera | Front cam |
| `/camera_wrist/image_raw` | Image | usb_cam | Wrist cam |

---

## 5. Cài Đặt Lần Đầu

### Bước 1: Build workspace
```bash
cd ~/ur5_teleop_vive
colcon build --packages-select ur5_teleop_vive
source install/setup.bash
```

### Bước 2: Source mỗi terminal
**Mọi terminal** chạy node phải source workspace trước:
```bash
source /home/khanh/ur5_teleop_vive/install/setup.bash
```

Hoặc thêm vào `~/.bashrc` để tự động:
```bash
echo "source /home/khanh/ur5_teleop_vive/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### Bước 3: Kiểm tra robot kết nối
```bash
ping 192.168.1.1
```
Nếu OK → trên Polyscope: vào "Installation" → "URCaps" → enable External Control.

### Bước 4: Kiểm tra SteamVR
- Mở SteamVR
- Cắm headset (để init), sau đó có thể tắt
- Bật tracker (long press button đến khi đèn xanh)
- Lighthouses → SteamVR status phải show 2 lighthouse xanh + tracker xanh

---

## 6. Kiểm Tra Phần Cứng

### Test camera

**Tìm device path:**
```bash
v4l2-ctl --list-devices
```

Output mẫu:
```
HD Webcam: HD Webcam (laptop)     /dev/video0, /dev/video1
Intel RealSense D435I              /dev/video2-7 (nhiều subdevice)
C922 Pro Stream Webcam             /dev/video8, /dev/video9
```

**Test Realsense:**
```bash
realsense-viewer
```
→ Bật RGB Camera, xem có ra ảnh không.

**Test webcam USB (C922):**
```bash
ffplay /dev/video8
```

**Check USB speed cho Realsense:**
```bash
ls /sys/bus/usb/devices/ | xargs -I{} sh -c 'echo -n "{}: "; cat /sys/bus/usb/devices/{}/speed 2>/dev/null; echo ""' | grep -v "^$"
```
Realsense phải ở Bus có tốc độ 5000M (USB 3.0) hoặc cao hơn. Nếu thấy 480M (USB 2.0) → đổi cáp/cổng.

### Test ROS topics đầy đủ

Sau khi chạy đủ 6 terminal trước (T1-T6), check:
```bash
ros2 topic list | grep -E "camera|ur_|controller"
```

Phải thấy đầy đủ:
```
/camera/camera/color/image_raw
/camera_wrist/image_raw
/right_controller_as_posestamped
/ur_actual_pose
/ur_joint_states
/ur_target_pose
/vive_right
```

Check Hz và delay:
```bash
ros2 topic hz /camera/camera/color/image_raw --window 10
ros2 topic hz /camera_wrist/image_raw --window 10
ros2 topic delay /camera/camera/color/image_raw --window 10
```

Mục tiêu: ~15-30 Hz, delay < 1 giây.

---

## 7. Calibration (Hiệu Chuẩn)

### Tại sao cần calibration?

Vive frame và Robot frame là 2 hệ tọa độ khác nhau. Calibration tính ra ma trận 4×4 biến đổi từ Vive space → Robot space.

### Cách 1: Calibration đơn giản (chỉ yaw quanh Z)

Dùng khi setup tracker và robot ở cùng mặt phẳng:
```bash
python3 calib_manual.py
```

Hướng dẫn:
1. Gắn tracker lên flange robot
2. Dùng Teach Pendant di chuyển robot theo trục +X khoảng 50cm qua 6 điểm
3. Script tự tính góc yaw
4. Output: `world_alignment_matrix.txt`

### Cách 2: Full 4×4 calibration (chính xác hơn)

```bash
python3 calib_4x4.py
```

1. Gắn tracker lên flange robot
2. Di chuyển robot đến 8 điểm RẢI ĐỀU TRONG KHÔNG GIAN 3D (không thẳng hàng, không cùng mặt phẳng)
3. Mỗi điểm cách nhau ít nhất 10cm
4. Script dùng Kabsch SVD → tính ma trận 4×4 đầy đủ
5. Output: `world_alignment_matrix.txt` + RMSE

**RMSE chấp nhận được:**
- `< 5mm` → Excellent
- `< 10mm` → Good
- `< 20mm` → Acceptable (lighthouse setup có thể chưa tối ưu)
- `> 20mm` → Cần thu lại, kiểm tra tracker không bị che

### Lưu ý
- Calibration chỉ cần làm 1 lần, miễn là lighthouses và robot không bị di chuyển
- Nếu di chuyển bất kỳ thiết bị nào → calibrate lại

---

## 8. Workflow Thu Data

### Khởi động pipeline (7 terminal)

**Mỗi terminal phải `source` workspace trước.**

**Terminal 1 — Vive bridge:**
```bash
python3 vive_tf_and_joy_ros2.py
```

**Terminal 2 — TF converter:**
```bash
python3 frame_as_posestamped_ros2.py
```

**Terminal 3 — Teleop logic:**
```bash
python3 vive_ur5_teleop_params.py
```

**Terminal 4 — Robot control:**
```bash
python3 ur_follow_using_class_ros2.py
```
Robot sẽ tự di chuyển về home position, đợi log:
```
SYSTEM READY — RTDE @ 100Hz, state 30Hz
```

**Terminal 5 — Realsense front:**
```bash
ros2 run realsense2_camera realsense2_camera_node --ros-args \
  -p enable_depth:=false \
  -p enable_infra1:=false \
  -p enable_infra2:=false \
  -p enable_gyro:=false \
  -p enable_accel:=false \
  -p rgb_camera.color_profile:="640x480x30"
```

**Terminal 6 — Webcam wrist (đổi `/dev/videoX` cho đúng):**
```bash
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:=/dev/video8 \
  -p pixel_format:=mjpeg2rgb \
  -p image_width:=640 \
  -p image_height:=480 \
  -p framerate:=30.0 \
  -r image_raw:=/camera_wrist/image_raw
```

**Terminal 7 — Recorder:**
```bash
python3 record_all.py --task pick_cube --fps 10
```

### Điều khiển teleop

**Phím nóng (focus vào terminal vive_tf_and_joy_ros2):**
- **Home (phím Home)** — Set origin: Robot ghi nhớ vị trí TCP hiện tại làm gốc. Vive di chuyển sẽ tính delta từ gốc này.
- **Ctrl_R** — Toggle ON/OFF teleop:
  - Lần nhấn 1: **bật** teleop, ghi baseline orientation. Từ giờ robot follow Vive.
  - Lần nhấn 2: **tắt** teleop, robot dừng.
- Khi bật lần đầu, log sẽ in: `🎯 Baseline Vive orientation captured`

### Workflow ghi episode

Trong GUI của `record_all.py`:

```
1. Đợi cả 2 camera viền XANH, 3 indicator ● sáng (joints, actual, target)

2. Đưa robot về vị trí bắt đầu (Home), đặt vật cần gắp lên bàn

3. Cầm Vive tracker ở tư thế thoải mái

4. Nhấn Ctrl_R → robot bắt đầu follow Vive

5. SPACE → bắt đầu ghi episode
   GUI hiện: 🔴 REC  ep_NNNN  frame=X  t=Xs

6. Thực hiện demo:
   - Di chuyển Vive → robot gắp vật
   - (Khi có gripper) Đóng gripper khi tới vật
   - Di chuyển đến target → mở gripper thả vật

7. Kết thúc:
   - S → lưu SUCCESS (demo OK)
   - F → lưu FAIL (demo lỗi, vẫn ghi để lọc khi train)
   - SPACE → dừng nhưng chưa lưu (đợi S hoặc F)

8. Nhấn Ctrl_R để tắt teleop, robot dừng

9. Lặp lại từ bước 2 cho episode tiếp theo
```

### Số lượng episode khuyến nghị

| Mức | Số episode | Dùng để |
|---|---|---|
| **Test pipeline** | 1-5 | Verify code chạy đúng |
| **Behavioral cloning đơn giản** | 20-50 | Test khái niệm |
| **Pi0.5 fine-tune cơ bản** | 50-100 | Có thể train ra |
| **Pi0.5 fine-tune chất lượng** | 200-500 | Production-ready |
| **Pi0.5 fine-tune robust** | 1000+ | Generalization tốt |

### Đa dạng hóa episode

Để Pi0.5 generalize tốt:
- Đặt vật ở **vị trí khác nhau** mỗi episode (không cố định)
- **Hướng vật** khác nhau (xoay)
- **Ánh sáng** thay đổi (sáng/tối khác nhau)
- Camera **không được di chuyển** giữa các episode!

---

## 9. Định Dạng Dataset

### Cấu trúc thư mục

```
dataset/pick_cube/
└── episodes/
    ├── ep_0000/
    │   ├── images/
    │   │   ├── front/      ← Realsense (top-down)
    │   │   │   ├── 000000.jpg
    │   │   │   ├── 000001.jpg
    │   │   │   └── ...
    │   │   └── wrist/      ← C922 (gripper view)
    │   │       ├── 000000.jpg
    │   │       └── ...
    │   ├── frames.jsonl   ← Data từng frame
    │   └── meta.json      ← Metadata episode
    ├── ep_0001/
    └── ...
```

### Format `frames.jsonl` (1 dòng = 1 frame)

```json
{
  "frame": 0,
  "t": 0.034,
  "joints": [-0.012, -1.535, 1.523, -1.420, -1.551, 2.924],
  "actual": {
    "x": -0.296, "y": -0.111, "z": 0.305,
    "roll": -2.380, "pitch": 1.934, "yaw": -0.190
  },
  "target": {
    "x": -0.298, "y": -0.111, "z": 0.304,
    "qx": 0.873, "qy": -0.484, "qz": 0.044, "qw": -0.035
  },
  "front": "images/front/000000.jpg",
  "wrist": "images/wrist/000000.jpg"
}
```

**Giải thích:**
- `frame` — index frame (0, 1, 2, ...)
- `t` — giây từ lúc bắt đầu episode (đều nhau theo FPS)
- `joints` — 6 joint positions (rad), thứ tự: shoulder_pan → wrist_3
- `actual` — TCP pose thực tế (forward kinematics của joints)
- `target` — Pose mong muốn từ Vive (đây là **action label** cho Pi0.5)
- `front`, `wrist` — relative paths đến ảnh JPG 224×224

### Format `meta.json`

```json
{
  "ep": 0,
  "task": "pick_cube",
  "success": true,
  "n_frames": 230,
  "fps_target": 10,
  "fps_actual": 10.03,
  "duration_s": 22.93,
  "image_size": [224, 224],
  "robot": "UR3",
  "fields": {...},
  "topics": {...},
  "saved_at": "2026-05-17 15:31:21"
}
```

### Đảm bảo sync

`t` tăng **chính xác 1/FPS giây** mỗi frame (vd: FPS=10 → t=0.1, 0.2, 0.3...).

Verify nhanh:
```bash
head -5 dataset/pick_cube/episodes/ep_0000/frames.jsonl | \
  python3 -c "
import sys, json
for line in sys.stdin:
    d = json.loads(line)
    print(f't={d[\"t\"]:.3f}s  frame={d[\"frame\"]}')
"
```

---

## 10. Troubleshooting

### Camera không thấy data
**Symptom:** `record_all.py` hiện viền đỏ "NO DATA"

**Fix:**
- Check camera node đang chạy: `ros2 topic list | grep camera`
- Check device đúng: `v4l2-ctl --list-devices` (số `/dev/videoX` thay đổi mỗi lần cắm USB!)
- Restart camera node với device đúng

### `actual` luôn null trong frames.jsonl
**Symptom:** Trường `actual: null` mọi frame, dù `ur_follow` chạy

**Nguyên nhân:** Workspace `ur5_teleop_vive` chưa được source trong terminal chạy recorder → không import được kiểu `Xyzrpy`

**Fix:**
```bash
source /home/khanh/ur5_teleop_vive/install/setup.bash
# Rồi mới chạy record_all.py
```

Test:
```bash
ros2 topic echo /ur_actual_pose --once
```
Nếu báo `invalid` → chưa source. Nếu ra data x,y,z → OK.

### Realsense delay cao (~1-2 giây)
**Symptom:** GUI hiển thị trễ, `ros2 topic delay` báo > 1s

**Đây là vấn đề cố hữu của Realsense ROS driver** — không fix được hoàn toàn.

**Tuy nhiên không ảnh hưởng dataset** vì `record_all.py` dùng `time.time()` làm master clock, không dùng timestamp của message.

Giảm tải bằng cách tắt depth/IR/IMU:
```bash
ros2 run realsense2_camera realsense2_camera_node --ros-args \
  -p enable_depth:=false -p enable_infra1:=false -p enable_infra2:=false \
  -p enable_gyro:=false -p enable_accel:=false \
  -p rgb_camera.color_profile:="640x480x30"
```

### Realsense chạy 3fps thay vì 30fps
**Symptom:** `ros2 topic hz` báo 3 Hz cho `/camera/camera/color/image_raw`

**Nguyên nhân:** USB 2.0 hoặc bandwidth bị giới hạn

**Fix:**
- Check USB speed: `lsusb -t` → tìm Realsense, phải có 5000M (USB 3.0)
- Đổi cáp USB-C → USB-A 3.0 (cáp gốc đi kèm Realsense có thể là USB 2.0)
- Cắm sang cổng USB 3.0 trên laptop (cổng xanh, hoặc check `lsusb -t`)

### Robot không di chuyển khi nhấn Ctrl_R
**Symptom:** Vive di chuyển nhưng robot đứng yên

**Check theo thứ tự:**
1. `ur_follow_using_class_ros2.py` có log `SYSTEM READY`?
2. `vive_ur5_teleop_params.py` có log `Baseline captured` khi nhấn Ctrl_R?
3. Đã nhấn **Home** trước Ctrl_R chưa? (Home set origin, Ctrl_R bật teleop)
4. Tracker có xanh trong SteamVR không?

### "RTDEControlInterface: RTDE control script is not running"
**Nguyên nhân:** UR3 đã dừng RTDE script (do safety/protective stop hoặc reset)

**Fix:** Code đã auto-reconnect. Nếu vẫn lỗi:
- Trên Polyscope: nhấn nút **Play** (icon ▶) ở thanh dưới
- Hoặc Restart `ur_follow` terminal

### Quaternion target có giá trị `[1, 0, 0, 0]` đứng yên
**Nguyên nhân:** Bạn chưa nhấn Ctrl_R hoặc Vive tracker không tracking

**Fix:** Trong SteamVR phải thấy tracker xanh; nhấn Ctrl_R để bật teleop

### GUI Wayland không hiện cửa sổ
**Fix:** Set env trước khi chạy
```bash
QT_QPA_PLATFORM=xcb python3 record_all.py --task pick_cube --fps 10
```

(Code đã set sẵn `os.environ['QT_QPA_PLATFORM'] = 'xcb'` ở đầu file.)

---

## 11. Roadmap

### Đã hoàn thành ✅
- [x] Pipeline teleop Vive → UR3 ổn định
- [x] Calibration 4×4 đạt RMSE ~16mm
- [x] 2 camera setup (Realsense + C922)
- [x] Dataset recorder với GUI + sync chuẩn 10Hz
- [x] Format compatible với Pi0.5 (224×224 RGB, joints, pose)

### Cần làm tiếp 🚧
- [ ] **Lắp gripper** (Robotiq 2F-85 hoặc DIY) — Pi0.5 cần gripper open/close action
- [ ] **Thêm `gripper_state`** vào frames.jsonl (0=open, 1=closed)
- [ ] **Thêm `language_instruction`** vào meta.json (vd: "pick up the red cube")
- [ ] **Convert script** sang format LeRobot v2 (HuggingFace dataset)
- [ ] **Thu 100+ episodes** với đa dạng vị trí vật
- [ ] **Fine-tune Pi0.5** (cần GPU ≥ RTX 3090 hoặc thuê A100 Colab)

### Reference
- Pi0.5 paper: https://www.physicalintelligence.company
- openpi repo: https://github.com/Physical-Intelligence/openpi
- LeRobot: https://github.com/huggingface/lerobot

---

## 📝 License & Credit

Dự án này là thesis project tại Trường Đại học Bách Khoa Hà Nội. Code dựa trên:
- `URBasic` → migrated to `ur_rtde` (community-maintained)
- OpenVR Python bindings
- LeRobot dataset format

Liên hệ: khanh@... (cập nhật)

---

*README cập nhật: May 2026*
