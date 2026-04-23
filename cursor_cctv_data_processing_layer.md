# CCTV data processing layer
_Exported on 1/4/2026 at 19:18:32 GMT+5:30 from Cursor (2.2.44)_

---

**User**

@api-gateway-refactored.md 
let's talk about api gateway layer
how data will flow from cctv  to object detection model yolov8
YOLOv8 (Ultralytics)   🚗Object detection
OpenCV                    🎥 Video processing
Pandas & NumPy     📊Data handling  
Winsound                🔊Audio alerts
give me complete data processing layer for cctv data only
explain in simple words

---

**Cursor**


Reviewing the existing codebase to understand the current implementation, then outlining the CCTV-to-YOLOv8 data flow.



Outlining the CCTV-to-YOLOv8 data flow and a complete data processing layer for CCTV data.



## CCTV to YOLOv8 data flow — simple explanation

### The journey: CCTV camera → YOLOv8 model

```
CCTV Camera (RTSP Stream)
    ↓
Video Gateway (Receives stream)
    ↓
OpenCV (Extracts frames)
    ↓
YOLOv8 Model (Detects objects)
    ↓
Pandas/NumPy (Processes results)
    ↓
Output (Tracking, Speed, CSV)
```

---

## Complete data processing layer for CCTV

### Step 1: CCTV camera sends video stream
- CCTV sends video over RTSP (like a live TV stream)
- Example: `rtsp://192.168.1.100:554/stream`
- Sends 25–30 frames per second

### Step 2: Video gateway receives stream
- Gateway connects to the RTSP stream
- Uses FFmpeg to convert RTSP to frames
- Extracts frames one by one

### Step 3: OpenCV processes frames
- OpenCV reads each frame as an image
- Resizes if needed (e.g., 1020x500)
- Converts to a format YOLOv8 can use

### Step 4: YOLOv8 detects objects
- YOLOv8 analyzes the frame
- Finds vehicles (cars, trucks, buses, etc.)
- Returns bounding boxes and class labels

### Step 5: Pandas/NumPy organize data
- Pandas stores detection results in a DataFrame
- NumPy handles numerical calculations
- Data is structured for tracking

### Step 6: Tracker follows vehicles
- Assigns unique IDs to each vehicle
- Tracks movement across frames
- Calculates speed

### Step 7: Results saved
- CSV file with vehicle details
- Annotated video with bounding boxes
- Real-time alerts (optional)

---

## Complete code structure

```python
# ============================================================================
# COMPLETE CCTV DATA PROCESSING LAYER
# ============================================================================

import cv2
import pandas as pd
import numpy as np
from ultralytics import YOLO
from tracker import Tracker
import time
import csv
from datetime import datetime

class CCTVDataProcessor:
    """
    Complete data processing layer for CCTV streams
    Handles: Video → OpenCV → YOLOv8 → Pandas/NumPy → Output
    """
    
    def __init__(self, rtsp_url, model_path='yolov8s.pt'):
        """
        Initialize the processor
        
        Args:
            rtsp_url: RTSP stream URL from CCTV camera
            model_path: Path to YOLOv8 model file
        """
        # Step 1: Load YOLOv8 model
        print("Loading YOLOv8 model...")
        self.model = YOLO(model_path)
        
        # Step 2: Connect to CCTV stream using OpenCV
        print(f"Connecting to CCTV stream: {rtsp_url}")
        self.cap = cv2.VideoCapture(rtsp_url)
        
        # Step 3: Get video properties
        self.fps = int(self.cap.get(cv2.CAP_PROP_FPS)) or 25
        self.width = int(self.cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        self.height = int(self.cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        
        # Step 4: Initialize tracker
        self.tracker = Tracker(fps=self.fps, calibration_factor=0.05)
        
        # Step 5: Load class names (for YOLOv8)
        with open('coco.txt', 'r') as f:
            self.class_list = f.read().split("\n")
        
        # Step 6: Setup data storage (Pandas will use this)
        self.detection_data = []
        
        # Step 7: Setup CSV writer
        self.csv_file = open('cctv_detections.csv', 'w', newline='')
        self.csv_writer = csv.writer(self.csv_file)
        self.csv_writer.writerow([
            "Timestamp", "Frame_Number", "Object_ID", "Class", 
            "Bbox_X1", "Bbox_Y1", "Bbox_X2", "Bbox_Y2", 
            "Speed_kmh", "Confidence"
        ])
        
        print("CCTV Data Processor initialized!")
    
    def process_frame(self, frame, frame_number):
        """
        Process a single frame through the pipeline
        
        Pipeline: Frame → YOLOv8 → Pandas → NumPy → Tracker
        """
        # ====================================================================
        # STEP 1: OpenCV Preprocessing
        # ====================================================================
        # Resize frame for faster processing
        frame_resized = cv2.resize(frame, (1020, 500))
        
        # ====================================================================
        # STEP 2: YOLOv8 Object Detection
        # ====================================================================
        # YOLOv8 analyzes the frame and detects objects
        results = self.model.predict(
            frame_resized, 
            conf=0.2,  # Confidence threshold (20%)
            verbose=False
        )
        
        # Extract detection boxes from YOLOv8 results
        boxes = results[0].boxes.data  # This is a NumPy array
        
        # ====================================================================
        # STEP 3: Pandas Data Organization
        # ====================================================================
        # Convert NumPy array to Pandas DataFrame for easy manipulation
        df = pd.DataFrame(boxes).astype("float")
        
        # DataFrame columns: [x1, y1, x2, y2, confidence, class_id]
        
        # ====================================================================
        # STEP 4: Filter and Process Detections
        # ====================================================================
        detections = []
        allowed_classes = ['car', 'truck', 'bus', 'motorbike', 'bicycle']
        
        for index, row in df.iterrows():
            x1 = int(row[0])
            y1 = int(row[1])
            x2 = int(row[2])
            y2 = int(row[3])
            confidence = float(row[4])
            class_id = int(row[5])
            class_name = self.class_list[class_id]
            
            # Only process vehicles
            if class_name in allowed_classes:
                detections.append([x1, y1, x2, y2, class_name, confidence])
        
        # ====================================================================
        # STEP 5: Tracker Updates (uses NumPy for calculations)
        # ====================================================================
        # Update tracker with bounding boxes
        bbox_coords = [[d[0], d[1], d[2], d[3]] for d in detections]
        tracked_objects = self.tracker.update(bbox_coords)
        
        # tracked_objects format: [x1, y1, x2, y2, object_id, speed_kmh]
        
        # ====================================================================
        # STEP 6: Save Results to CSV (using Pandas/CSV)
        # ====================================================================
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        for i, tracked in enumerate(tracked_objects):
            x1, y1, x2, y2, obj_id, speed = tracked
            class_name = detections[i][4] if i < len(detections) else "unknown"
            confidence = detections[i][5] if i < len(detections) else 0.0
            
            # Write to CSV
            self.csv_writer.writerow([
                timestamp, frame_number, obj_id, class_name,
                x1, y1, x2, y2, speed, confidence
            ])
        
        # ====================================================================
        # STEP 7: Draw Results on Frame (OpenCV visualization)
        # ====================================================================
        annotated_frame = self.draw_detections(frame_resized, tracked_objects, detections)
        
        return annotated_frame, tracked_objects
    
    def draw_detections(self, frame, tracked_objects, detections):
        """
        Draw bounding boxes and labels on frame using OpenCV
        """
        for i, tracked in enumerate(tracked_objects):
            x1, y1, x2, y2, obj_id, speed = tracked
            
            # Draw bounding box
            cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
            
            # Draw object ID and speed
            label = f"ID:{obj_id} {speed:.1f}km/h"
            cv2.putText(frame, label, (x1, y1-10), 
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 2)
        
        return frame
    
    def run(self):
        """
        Main processing loop
        """
        frame_count = 0
        
        print("Starting CCTV stream processing...")
        
        while True:
            # Read frame from CCTV stream
            ret, frame = self.cap.read()
            
            if not ret:
                print("Failed to read frame. Reconnecting...")
                time.sleep(2)
                self.cap.release()
                self.cap = cv2.VideoCapture(self.rtsp_url)
                continue
            
            frame_count += 1
            
            # Process every 3rd frame for performance
            if frame_count % 3 != 0:
                continue
            
            # Process frame through complete pipeline
            annotated_frame, tracked_objects = self.process_frame(frame, frame_count)
            
            # Display frame
            cv2.imshow("CCTV Processing", annotated_frame)
            
            # Print statistics
            if frame_count % 30 == 0:  # Every 30 frames
                print(f"Processed {frame_count} frames. "
                      f"Active objects: {len(tracked_objects)}")
            
            # Exit on ESC key
            if cv2.waitKey(1) & 0xFF == 27:
                break
        
        # Cleanup
        self.cleanup()
    
    def cleanup(self):
        """Close all resources"""
        self.cap.release()
        self.csv_file.close()
        cv2.destroyAllWindows()
        print("Processing complete. Data saved to cctv_detections.csv")


# ============================================================================
# USAGE EXAMPLE
# ============================================================================

if __name__ == "__main__":
    # CCTV RTSP stream URL
    rtsp_url = "rtsp://192.168.1.100:554/stream"
    
    # Create processor
    processor = CCTVDataProcessor(rtsp_url)
    
    # Start processing
    processor.run()
```

---

## Data flow diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CCTV CAMERA                              │
│              (RTSP Stream @ 25-30 FPS)                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              VIDEO GATEWAY (FFmpeg/OpenCV)                  │
│  • Connects to RTSP stream                                  │
│  • Extracts frames from video stream                        │
│  • Converts to OpenCV format (NumPy array)                 │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              OPENCV FRAME PROCESSING                        │
│  • Reads frame as NumPy array                              │
│  • Resizes: (1920x1080) → (1020x500)                       │
│  • Converts color format if needed                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              YOLOV8 OBJECT DETECTION                        │
│  • Input: NumPy array (frame image)                        │
│  • Output: Bounding boxes + classes + confidence           │
│  • Detects: cars, trucks, buses, etc.                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              PANDAS DATA ORGANIZATION                       │
│  • Converts YOLOv8 results to DataFrame                    │
│  • Filters by vehicle classes                              │
│  • Organizes: [x1, y1, x2, y2, class, confidence]         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              NUMPY CALCULATIONS                            │
│  • Distance calculations (pixel → meters)                  │
│  • Speed calculations (distance / time)                     │
│  • Array operations for tracking                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              TRACKER (Vehicle Tracking)                     │
│  • Assigns unique IDs to vehicles                          │
│  • Tracks movement across frames                           │
│  • Calculates speed in km/h                                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              OUTPUT (CSV + Annotated Video)                 │
│  • CSV: Timestamp, ID, Class, Speed, Bbox                  │
│  • Video: Frame with bounding boxes + labels               │
└─────────────────────────────────────────────────────────────┘
```

---

## Simple explanation of each component

### 1. CCTV camera (RTSP stream)
- Sends live video over the network
- Like a security camera feed

### 2. Video gateway (FFmpeg/OpenCV)
- Connects to the camera
- Extracts individual frames (images) from the video

### 3. OpenCV
- Reads each frame as an image
- Resizes and prepares it for YOLOv8

### 4. YOLOv8
- Analyzes the frame
- Finds vehicles and returns boxes and labels

### 5. Pandas
- Organizes detection results in a table
- Filters and structures the data

### 6. NumPy
- Handles math (distances, speeds)
- Works with arrays efficiently

### 7. Tracker
- Follows the same vehicle across frames
- Assigns IDs and computes speed

### 8. Output
- Saves results to CSV
- Draws boxes on video for visualization

---

## Key points

1. Real-time processing: processes frames as they arrive
2. Batch processing: can process every Nth frame for speed
3. Data pipeline: each step feeds into the next
4. Storage: results saved to CSV for analysis
5. Visualization: annotated video shows detections

This setup processes CCTV streams, detects vehicles with YOLOv8, tracks them, and saves the results.

```plaintext
CCTV Camera (RTSP Stream)
    ↓
Video Gateway (Receives stream)
    ↓
OpenCV (Extracts frames)
    ↓
YOLOv8 Model (Detects objects)
    ↓
Pandas/NumPy (Processes results)
    ↓
Output (Tracking, Speed, CSV)
```

```python
# ============================================================================
# COMPLETE CCTV DATA PROCESSING LAYER
# ============================================================================

import cv2
import pandas as pd
import numpy as np
from ultralytics import YOLO
from tracker import Tracker
import time
import csv
from datetime import datetime

class CCTVDataProcessor:
    """
    Complete data processing layer for CCTV streams
    Handles: Video → OpenCV → YOLOv8 → Pandas/NumPy → Output
    """
    
    def __init__(self, rtsp_url, model_path='yolov8s.pt'):
        """
        Initialize the processor
        
        Args:
            rtsp_url: RTSP stream URL from CCTV camera
            model_path: Path to YOLOv8 model file
        """
        # Step 1: Load YOLOv8 model
        print("Loading YOLOv8 model...")
        self.model = YOLO(model_path)
        
        # Step 2: Connect to CCTV stream using OpenCV
        print(f"Connecting to CCTV stream: {rtsp_url}")
        self.cap = cv2.VideoCapture(rtsp_url)
        
        # Step 3: Get video properties
        self.fps = int(self.cap.get(cv2.CAP_PROP_FPS)) or 25
        self.width = int(self.cap.get(cv2.CAP_PROP_FRAME_WIDTH))
        self.height = int(self.cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
        
        # Step 4: Initialize tracker
        self.tracker = Tracker(fps=self.fps, calibration_factor=0.05)
        
        # Step 5: Load class names (for YOLOv8)
        with open('coco.txt', 'r') as f:
            self.class_list = f.read().split("\n")
        
        # Step 6: Setup data storage (Pandas will use this)
        self.detection_data = []
        
        # Step 7: Setup CSV writer
        self.csv_file = open('cctv_detections.csv', 'w', newline='')
        self.csv_writer = csv.writer(self.csv_file)
        self.csv_writer.writerow([
            "Timestamp", "Frame_Number", "Object_ID", "Class", 
            "Bbox_X1", "Bbox_Y1", "Bbox_X2", "Bbox_Y2", 
            "Speed_kmh", "Confidence"
        ])
        
        print("CCTV Data Processor initialized!")
    
    def process_frame(self, frame, frame_number):
        """
        Process a single frame through the pipeline
        
        Pipeline: Frame → YOLOv8 → Pandas → NumPy → Tracker
        """
        # ====================================================================
        # STEP 1: OpenCV Preprocessing
        # ====================================================================
        # Resize frame for faster processing
        frame_resized = cv2.resize(frame, (1020, 500))
        
        # ====================================================================
        # STEP 2: YOLOv8 Object Detection
        # ====================================================================
        # YOLOv8 analyzes the frame and detects objects
        results = self.model.predict(
            frame_resized, 
            conf=0.2,  # Confidence threshold (20%)
            verbose=False
        )
        
        # Extract detection boxes from YOLOv8 results
        boxes = results[0].boxes.data  # This is a NumPy array
        
        # ====================================================================
        # STEP 3: Pandas Data Organization
        # ====================================================================
        # Convert NumPy array to Pandas DataFrame for easy manipulation
        df = pd.DataFrame(boxes).astype("float")
        
        # DataFrame columns: [x1, y1, x2, y2, confidence, class_id]
        
        # ====================================================================
        # STEP 4: Filter and Process Detections
        # ====================================================================
        detections = []
        allowed_classes = ['car', 'truck', 'bus', 'motorbike', 'bicycle']
        
        for index, row in df.iterrows():
            x1 = int(row[0])
            y1 = int(row[1])
            x2 = int(row[2])
            y2 = int(row[3])
            confidence = float(row[4])
            class_id = int(row[5])
            class_name = self.class_list[class_id]
            
            # Only process vehicles
            if class_name in allowed_classes:
                detections.append([x1, y1, x2, y2, class_name, confidence])
        
        # ====================================================================
        # STEP 5: Tracker Updates (uses NumPy for calculations)
        # ====================================================================
        # Update tracker with bounding boxes
        bbox_coords = [[d[0], d[1], d[2], d[3]] for d in detections]
        tracked_objects = self.tracker.update(bbox_coords)
        
        # tracked_objects format: [x1, y1, x2, y2, object_id, speed_kmh]
        
        # ====================================================================
        # STEP 6: Save Results to CSV (using Pandas/CSV)
        # ====================================================================
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        for i, tracked in enumerate(tracked_objects):
            x1, y1, x2, y2, obj_id, speed = tracked
            class_name = detections[i][4] if i < len(detections) else "unknown"
            confidence = detections[i][5] if i < len(detections) else 0.0
            
            # Write to CSV
            self.csv_writer.writerow([
                timestamp, frame_number, obj_id, class_name,
                x1, y1, x2, y2, speed, confidence
            ])
        
        # ====================================================================
        # STEP 7: Draw Results on Frame (OpenCV visualization)
        # ====================================================================
        annotated_frame = self.draw_detections(frame_resized, tracked_objects, detections)
        
        return annotated_frame, tracked_objects
    
    def draw_detections(self, frame, tracked_objects, detections):
        """
        Draw bounding boxes and labels on frame using OpenCV
        """
        for i, tracked in enumerate(tracked_objects):
            x1, y1, x2, y2, obj_id, speed = tracked
            
            # Draw bounding box
            cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)
            
            # Draw object ID and speed
            label = f"ID:{obj_id} {speed:.1f}km/h"
            cv2.putText(frame, label, (x1, y1-10), 
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 255), 2)
        
        return frame
    
    def run(self):
        """
        Main processing loop
        """
        frame_count = 0
        
        print("Starting CCTV stream processing...")
        
        while True:
            # Read frame from CCTV stream
            ret, frame = self.cap.read()
            
            if not ret:
                print("Failed to read frame. Reconnecting...")
                time.sleep(2)
                self.cap.release()
                self.cap = cv2.VideoCapture(self.rtsp_url)
                continue
            
            frame_count += 1
            
            # Process every 3rd frame for performance
            if frame_count % 3 != 0:
                continue
            
            # Process frame through complete pipeline
            annotated_frame, tracked_objects = self.process_frame(frame, frame_count)
            
            # Display frame
            cv2.imshow("CCTV Processing", annotated_frame)
            
            # Print statistics
            if frame_count % 30 == 0:  # Every 30 frames
                print(f"Processed {frame_count} frames. "
                      f"Active objects: {len(tracked_objects)}")
            
            # Exit on ESC key
            if cv2.waitKey(1) & 0xFF == 27:
                break
        
        # Cleanup
        self.cleanup()
    
    def cleanup(self):
        """Close all resources"""
        self.cap.release()
        self.csv_file.close()
        cv2.destroyAllWindows()
        print("Processing complete. Data saved to cctv_detections.csv")


# ============================================================================
# USAGE EXAMPLE
# ============================================================================

if __name__ == "__main__":
    # CCTV RTSP stream URL
    rtsp_url = "rtsp://192.168.1.100:554/stream"
    
    # Create processor
    processor = CCTVDataProcessor(rtsp_url)
    
    # Start processing
    processor.run()
```

```plaintext
┌─────────────────────────────────────────────────────────────┐
│                    CCTV CAMERA                              │
│              (RTSP Stream @ 25-30 FPS)                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              VIDEO GATEWAY (FFmpeg/OpenCV)                  │
│  • Connects to RTSP stream                                  │
│  • Extracts frames from video stream                        │
│  • Converts to OpenCV format (NumPy array)                 │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              OPENCV FRAME PROCESSING                        │
│  • Reads frame as NumPy array                              │
│  • Resizes: (1920x1080) → (1020x500)                       │
│  • Converts color format if needed                         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              YOLOV8 OBJECT DETECTION                        │
│  • Input: NumPy array (frame image)                        │
│  • Output: Bounding boxes + classes + confidence           │
│  • Detects: cars, trucks, buses, etc.                      │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              PANDAS DATA ORGANIZATION                       │
│  • Converts YOLOv8 results to DataFrame                    │
│  • Filters by vehicle classes                              │
│  • Organizes: [x1, y1, x2, y2, class, confidence]         │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              NUMPY CALCULATIONS                            │
│  • Distance calculations (pixel → meters)                  │
│  • Speed calculations (distance / time)                     │
│  • Array operations for tracking                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              TRACKER (Vehicle Tracking)                     │
│  • Assigns unique IDs to vehicles                          │
│  • Tracks movement across frames                           │
│  • Calculates speed in km/h                                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              OUTPUT (CSV + Annotated Video)                 │
│  • CSV: Timestamp, ID, Class, Speed, Bbox                  │
│  • Video: Frame with bounding boxes + labels               │
└─────────────────────────────────────────────────────────────┘
```

