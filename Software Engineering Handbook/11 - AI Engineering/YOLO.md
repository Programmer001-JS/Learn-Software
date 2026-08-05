# YOLO

> **In one line —** an object detector fast enough for real-time video, because it looks at the whole image once instead of scanning it region by region.

| | |
|---|---|
| **Full name** | You Only Look Once |
| **Category** | Object Detection Model Family |
| **Architectural Layer** | Application |
| **Related notes** | [Computer Vision](Computer%20Vision.md) · [OpenCV](OpenCV.md) · [Deep Learning](Deep%20Learning.md) · [GPU Computing](GPU%20Computing.md) · [ONNX Runtime](ONNX%20Runtime.md) |

---

## 1. Short Definition

*What is it?*

YOLO is a family of object detection models that locate and classify multiple objects in one forward pass, returning bounding boxes with class labels and confidence scores.

---

## 2. Purpose

*What is its main purpose?*

Real-time detection. YOLO exists because earlier detectors were accurate but far too slow for video.

---

## 3. Problem

*What engineering problem does it solve?*

```text
EARLIER DETECTORS (R-CNN family)      YOLO
propose ~2,000 candidate regions      one pass over the whole image
run a classifier on each                  ↓
    ↓                                 predicts all boxes simultaneously
accurate but seconds per image            ↓
unusable for video                    tens to hundreds of frames per second
```

> [!IMPORTANT]
> The name is the idea: **one look**. Reframing detection as a single regression problem over a grid, rather than a search over regions, is what made real-time detection possible.

---

## 4. Architecture Position

```text
Camera / video stream
    ↓
OpenCV — decode, resize, colour convert
    ↓
YOLO model  (PyTorch, ONNX or TensorRT)
    ↓
Non-max suppression + confidence threshold
    ↓
Tracking (optional) → business logic
```

---

## 5. What it returns

```text
For each detection:
    class      "person"
    confidence 0.94
    box        x, y, width, height
```

Two postprocessing steps always follow:

```text
CONFIDENCE THRESHOLD   discard detections below, say, 0.4
NON-MAX SUPPRESSION    merge overlapping boxes for the same object
```

> [!TIP]
> **Both thresholds are business decisions.** A lower confidence threshold catches more objects and produces more false positives. For safety monitoring you want recall; for automated counting you want precision. Tune them against your actual cost of each error type, not to a default.

---

## 6. The version situation

> [!CAUTION]
> YOLO version numbers are not a straightforward progression. The original versions came from one research lineage; later numbered releases come from **different authors and different organisations**, with different licences. "YOLOv8 is newer than YOLOv7, so it is better" does not reliably hold.

**The practical consequence is licensing.** Several widely used YOLO implementations are **AGPL-licensed**, which has real implications for commercial use — AGPL can require you to publish source for network-accessible services built on it. Commercial licences are available for some.

> [!IMPORTANT]
> **Check the licence of the specific YOLO implementation before building a product on it.** This catches teams out regularly, and it is discovered late — usually at the point of a legal review before launch. Permissively licensed alternatives exist (including some Apache-2.0 detectors) if AGPL is unacceptable.

---

## 7. Model sizes

```text
nano / small     fastest, least accurate    → edge devices, high frame rates
medium           balanced                   → the usual starting point
large / extra    most accurate, slowest     → batch processing, high-value detection
```

> [!TIP]
> Start with the smallest model that meets your accuracy requirement, not the largest that fits. Inference cost is paid on every frame, forever; training cost is paid once.

---

## 8. Fine-tuning

```text
Pre-trained on a large general dataset (~80 common classes)
    ↓
Your classes are not in it: "cracked weld", "missing label"
    ↓
Fine-tune on a few hundred to a few thousand labelled images
    ↓
Working detector, hours of training on one GPU
```

Labelling means drawing bounding boxes — moderate effort, far less than segmentation masks.

---

## 9. Real World Example

- **Manufacturing** — detecting defects and missing components on a line.
- **Retail analytics** — counting people, monitoring shelf stock.
- **Traffic and parking** — vehicle counting, occupancy detection.
- **Safety compliance** — detecting whether personal protective equipment is worn.
- **Sports and broadcast** — player and ball tracking.

---

## 10. Detection is not tracking

```text
DETECTION   "there is a person in frame 1, and a person in frame 2"
TRACKING    "it is the SAME person"
```

> [!TIP]
> YOLO does not track. Counting unique people requires a tracker — **ByteTrack**, **DeepSORT** or similar — layered on top of the detections. Teams routinely discover this after building the detection half and finding their count increases by thirty every second.

---

## 11. Communication and Dependencies

- **A [GPU](GPU%20Computing.md)** for real-time video; CPU is viable for occasional images
- **[OpenCV](OpenCV.md)** for decoding and preprocessing
- **[ONNX Runtime](ONNX%20Runtime.md)** or **TensorRT** for optimised deployment
- **A tracker**, if identity across frames matters
- **Labelled data** in the expected annotation format

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use YOLO when you need to know **what** and **where**, in real time or at volume, and bounding boxes are sufficient.

> [!CAUTION]
> - **Not when you need exact shape** — that is segmentation
> - **Not for very small objects** in large images without tiling — a distant object may be a handful of pixels after resizing
> - **Not when classification alone would do** — a whole-image classifier is far cheaper
> - **Not without checking the licence**

---

## 13. Advantages and Disadvantages

**Advantages**
- Real-time on modest hardware
- Excellent accuracy-to-speed ratio
- Straightforward fine-tuning workflow
- Strong tooling and community
- Exports readily to ONNX and TensorRT for edge deployment

**Disadvantages**
- **Licensing complexity** across versions
- Struggles with very small or heavily overlapping objects
- Bounding boxes only — no shape information
- No tracking
- Version landscape is confusing
- Accuracy degrades sharply on conditions absent from training data

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Speed** | Tens to hundreds of FPS on GPU, depending on size |
| **CPU** | A few FPS — viable for images, not for video |
| **VRAM** | Modest for small models; batching increases it |
| **Input resolution** | The dominant cost factor — 640 is a common compromise |
| **TensorRT / ONNX** | Frequently 2–5× faster than the PyTorch model |

---

## 15. Security Considerations

> [!CAUTION]
> **Adversarial patches defeat detectors in the physical world.** Printed patterns worn or placed on an object can make it invisible to a detector or cause a misclassification — demonstrated repeatedly in research. Any security or safety system relying solely on detection is defeatable by someone who knows this.

- **Do not use detection as a sole security control** — combine it with other signals and human review
- **Detecting people is biometric-adjacent processing** with legal obligations in many jurisdictions
- **Video streams and stored frames are personal data** if people are identifiable
- **Untrusted video input** is decoded by native libraries — the same parsing risks as images
- **Model weights downloaded from the internet** are executable artifacts; verify their source

---

## 16. Mental Model

> [!NOTE]
> **Earlier detectors examined the image with a magnifying glass, region by region. YOLO glances at the whole scene at once.**
>
> The glance is far faster and slightly less careful — it may miss something small in the corner. For watching a video at thirty frames a second, the glance is the only option that exists.

---

## 17. Mini Architecture Diagram

```text
Video stream
    ↓
OpenCV: decode → resize 640×640 → RGB
    ↓
YOLO forward pass (batched, GPU)
    ↓
Raw boxes + class scores
    ↓
Confidence threshold → non-max suppression
    ↓
Tracker (ByteTrack) — assigns identities across frames
    ↓
Business logic: counting, alerting, recording
```

---

## 18. Complete Request Flow

Counting unique people entering a shop:

```text
Camera stream at 30 fps
    ↓
Process every 3rd frame                   ← 10 fps is enough for walking speed
    ↓
Decode, resize to 640×640, convert to RGB
    ↓
YOLO detects: 4 boxes classified "person", confidences 0.91–0.97
    ↓
Confidence threshold 0.5 applied
    ↓
Non-max suppression removes 1 duplicate box
    ↓
Tracker matches boxes to existing tracks
    ↓
A new track crossing the entry line → counter incremented
    ↓
Frames discarded immediately — not stored, avoiding a personal-data retention issue
    ↓
Only the count leaves the device
```

> [!TIP]
> Note the last two steps. **Processing at the edge and discarding frames** turns a personal-data problem into an anonymous counter — which is both cheaper and far easier to justify legally.

---

## 19. Key Takeaway

> [!IMPORTANT]
> YOLO gives real-time detection with bounding boxes — it does not track, it does not segment, and you must check the licence of the version you use before shipping a product.

---

## 20. Common Mistakes

- **Not checking the licence** until legal review
- **Expecting tracking** from a detector
- **Small objects lost** after resizing — tile the image instead
- **Default thresholds** rather than values tuned to your error costs
- **Processing every frame** when every third would do
- **Deploying the PyTorch model** rather than exporting to ONNX or TensorRT
- **Storing video frames** without considering the data protection consequences
- **Treating detection as a security control** on its own

---

## 21. Open Source Technologies

- **Ultralytics YOLO** — the most used implementation (check its licence)
- **YOLOX**, **RT-DETR**, **DETR** — alternative detectors with different licences
- **ByteTrack**, **DeepSORT** — tracking
- **ONNX Runtime**, **TensorRT**, **OpenVINO** — deployment
- **CVAT**, **Label Studio**, **Roboflow** — annotation

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check the licence of the YOLO implementation you intend to use, and whether it permits your use case.
- [ ] Run a detector on a video and observe what happens to your object count without a tracker.
- [ ] Compare inference speed for the PyTorch model and an ONNX export.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Stream → OpenCV preprocessing → YOLO → NMS → tracker → business logic
```

## 2. Request Flow

```text
Input       a video frame, resized and colour-converted
    ↓
Processing  one forward pass predicting all boxes, then NMS and thresholding
    ↓
Output      labelled bounding boxes with confidence scores
```

## 3. Real-World Usage

**Retail people-counting** is a common deployment: a camera at the entrance, detection at the edge, frames discarded immediately, and only an anonymous count transmitted. The architecture is driven as much by data protection as by bandwidth.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A real-time object detection model family |
| **Why does it exist?** | Because region-proposal detectors were too slow for video |
| **Where does it belong?** | Between image preprocessing and tracking or business logic |
| **When should I use it?** | Real-time "what and where" — with the licence checked first |
