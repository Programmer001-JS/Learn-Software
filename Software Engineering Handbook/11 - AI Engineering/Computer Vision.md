# Computer Vision

> **In one line —** teaching software to extract meaning from pixels, where the hard part is that a photograph of a cat and a photograph of the same cat in shade are entirely different numbers.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Application |
| **Sub-topics** | [OpenCV](OpenCV.md) · [YOLO](YOLO.md) · [OCR](OCR.md) |
| **Related notes** | [Deep Learning](Deep%20Learning.md) · [GPU Computing](GPU%20Computing.md) · [Model Serving](Model%20Serving.md) |

---

## 1. Short Definition

*What is it?*

Computer vision is the field concerned with extracting information from images and video — classifying, locating, segmenting, reading and tracking what appears in them.

---

## 2. Problem

*What engineering problem does it solve?*

```text
An image is a grid of numbers
    ↓
The same object under different light, angle, scale or occlusion
produces COMPLETELY DIFFERENT numbers
    ↓
Yet a human sees the same object instantly
    ↓
Bridging that gap is the entire field
```

---

## 3. The task types

> [!IMPORTANT]
> Choosing the right task is the most consequential decision, because it determines your labelling cost — which is usually the real budget.

```text
CLASSIFICATION    "this image contains a cat"
                  labelling: one tag per image           → cheap

DETECTION         "a cat, here, at these coordinates"
                  labelling: bounding boxes              → moderate

SEGMENTATION      "these exact pixels are cat"
                  labelling: pixel masks                 → EXPENSIVE

TRACKING          "the same cat across 300 frames"
                  labelling: identity across time        → very expensive

OCR               "the text in this image reads..."      → see OCR.md
```

```text
Do you need to know WHERE?     no → classification
Do you need exact shape?       no → detection (bounding boxes)
                              yes → segmentation
```

> [!TIP]
> Teams routinely choose segmentation when detection would do, then spend months labelling pixel masks. Ask what the downstream system actually consumes — a bounding box is usually enough.

---

## 4. Architecture Position

```text
Camera / upload
    ↓
Preprocessing  (resize, normalise, colour space)  ← must match training exactly
    ↓
Model  (CNN or vision transformer)
    ↓
Postprocessing  (thresholds, non-max suppression)
    ↓
Business logic
```

---

## 5. Classical vs deep learning

```text
CLASSICAL ([OpenCV](OpenCV.md))          DEEP LEARNING
edge detection, thresholding             learned features
template matching, contours                  ↓
    ↓                                    handles variation
fast, deterministic, no GPU              needs data and compute
brittle under lighting changes           robust
free                                     expensive
```

> [!IMPORTANT]
> **Classical techniques are not obsolete.** Reading a barcode, finding a document's edges, detecting motion, measuring a part against a fixed jig with controlled lighting — these are solved better by OpenCV than by a neural network, at a fraction of the cost and with deterministic behaviour.
>
> In practice, most production systems are **both**: classical preprocessing feeding a learned model.

---

## 6. The preprocessing trap

```text
TRAINING                              SERVING
resize to 224×224, bilinear           resize to 224×224, nearest neighbour
normalise with ImageNet mean/std      normalise to 0–1
RGB                                   BGR (OpenCV's default)
    ↓                                     ↓
        model accuracy collapses, with no error anywhere
```

> [!CAUTION]
> **This is the single most common production failure in computer vision**, and it produces no exception — just quietly worse results. OpenCV loads images as **BGR**, most models expect **RGB**, and the channel swap is silent. Share the preprocessing code between training and serving.

---

## 7. Data and labelling reality

```text
Model architecture       ██
Training technique       ███
DATA QUANTITY + QUALITY  ████████████
```

> [!TIP]
> **Augmentation is the cheapest improvement available.** Rotations, crops, brightness and contrast variation, and blur turn 500 images into effectively 5,000 — and it directly teaches invariance to the things that vary in production.
>
> Equally: augment for the conditions you will actually face. If the camera is fixed and the lighting is controlled, heavy rotation augmentation teaches nothing useful.

---

## 8. Real World Example

- **Manufacturing quality control** — detecting defects on a production line; the highest-value industrial application.
- **Document processing** — see [OCR](OCR.md).
- **Retail** — shelf monitoring, checkout-free stores.
- **Medical imaging** — assisting radiologists, under strict regulatory constraints.
- **Autonomous vehicles** — detection, segmentation and tracking simultaneously, in real time.

---

## 9. Video is not images

```text
30 fps × 1 camera = 30 inferences/second
    ↓  × 20 cameras
600 inferences/second, continuously, forever
```

> [!TIP]
> Video changes the economics completely. Standard mitigations: process every Nth frame, use motion detection to skip static frames, run a cheap model first and an expensive one only on candidates, and batch across cameras.

---

## 10. Communication and Dependencies

- **[OpenCV](OpenCV.md)** for image handling and classical operations
- **PyTorch** or **TensorFlow** with a pre-trained backbone
- **A [GPU](GPU%20Computing.md)** for real-time or high-volume work
- **[ONNX Runtime](ONNX%20Runtime.md)** or TensorRT for optimised inference
- **Labelling tooling** — a real project cost

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use computer vision where the information genuinely exists only in the image, and where a human currently looks at it. Start from a **pre-trained model plus fine-tuning**, never from scratch.

> [!CAUTION]
> - **If the data is available in structured form, use that** — reading a value from a screenshot when an API exists is a fragile solution to a solved problem
> - **Controlled conditions favour classical methods** — do not train a model to find a rectangle
> - **Perfect accuracy is not available** — design for human review of uncertain cases

---

## 12. Advantages and Disadvantages

**Advantages**
- Extracts information no other method can reach
- Transfer learning makes it viable with modest data
- Mature ecosystem and pre-trained models
- Automates work that currently requires a person to look

**Disadvantages**
- Labelling is expensive and slow
- Sensitive to conditions not represented in training data
- GPU cost for real-time and video workloads
- Poorly interpretable — "why did it miss that?" is hard to answer
- Adversarially fragile

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Latency** | Classical: microseconds. Deep: 10–100 ms per image |
| **GPU** | Required for real-time video; optional for batch |
| **Bandwidth** | Video is large — process at the edge where possible |
| **Batching** | Essential for throughput |
| **Resolution** | Cost scales with pixel count — do not process 4K to classify a scene |

---

## 14. Security Considerations

> [!CAUTION]
> **Vision systems that identify people are subject to strict and increasing legal regulation.** Biometric processing requires a lawful basis in most jurisdictions, and several restrict or ban facial recognition in public contexts outright. This is a legal question before it is a technical one.

- **Images contain metadata** — EXIF frequently includes GPS coordinates and device identifiers; strip it on upload
- **Adversarial patches** can defeat detectors in the physical world, printed on a sticker
- **Uploaded images are untrusted input** — malformed files have historically exploited image parsing libraries; validate and sandbox decoding
- **Retained training images are personal data** if they contain identifiable people
- **Bias in training data becomes discriminatory behaviour**, and in this domain it is well documented and legally consequential

---

## 15. Mental Model

> [!NOTE]
> **Classical vision is measuring with calipers; deep learning is recognising a face.**
>
> Calipers are exact, cheap and repeatable — as long as the part is positioned correctly every time. Recognition handles the messy real world where nothing is positioned correctly, and pays for it in training data and uncertainty.

---

## 16. Mini Architecture Diagram

```text
Camera / upload
    ↓
Validate and strip metadata
    ↓
Preprocess  (SHARED code with training)
    ↓
Cheap filter: motion or classical detection    ← skip work where possible
    ↓
Model inference (batched, on GPU)
    ↓
Postprocess: threshold, non-max suppression
    ↓
Confident?  → automated decision
Uncertain?  → human review queue
```

---

## 17. Complete Request Flow

```text
Image uploaded
    ↓
File validated; EXIF stripped
    ↓
Enqueued for background processing              ← not in the request path
    ↓
Worker loads it with OpenCV — BGR!
    ↓
Converted to RGB, resized, normalised — identical to training
    ↓
Batched with other pending images
    ↓
GPU inference → detections with confidence scores
    ↓
Non-max suppression removes overlapping boxes
    ↓
Confidence > 0.9 → accepted automatically
Confidence 0.5–0.9 → sent for human review
Confidence < 0.5 → discarded
    ↓
Human corrections feed back into the training set
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Choose the simplest task type your problem allows, keep preprocessing identical between training and serving, and use classical techniques wherever conditions are controlled.

---

## 19. Common Mistakes

- **Preprocessing mismatch**, especially BGR versus RGB — silent accuracy loss
- **Choosing segmentation** when detection would do, multiplying labelling cost
- **Training from scratch** instead of fine-tuning
- **No augmentation** on a small dataset
- **Processing full-resolution video** frame by frame
- **No human review path** for uncertain predictions
- **Ignoring the legal position** on biometric data
- **Not stripping EXIF** from uploaded images

---

## 20. Open Source Technologies

- **OpenCV** — classical vision and image handling
- **PyTorch**, **timm**, **Ultralytics YOLO** — models
- **Albumentations** — augmentation
- **CVAT**, **Label Studio** — labelling
- **ONNX Runtime**, **TensorRT**, **OpenVINO** — optimised inference
- **Segment Anything**, **Grounding DINO** — foundation models reducing labelling effort

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take a vision problem you have and decide the minimum task type that solves it.
- [ ] Compare your training and serving preprocessing line by line.
- [ ] Estimate the labelling cost for your dataset in person-hours before starting.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Camera → preprocessing → model → postprocessing → decision or human review
```

## 2. Request Flow

```text
Input       an image or video frame
    ↓
Processing  preprocessed consistently, inferred in a batch, thresholded
    ↓
Output      labels, boxes or masks — with uncertain cases escalated
```

## 3. Real-World Usage

**Manufacturing defect detection** is computer vision's most reliable business case: controlled lighting, a fixed camera, a clear definition of a defect, and a human currently doing the work. Those four conditions together are what make a vision project succeed.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Extracting structured information from images and video |
| **Why does it exist?** | Because some information exists only visually |
| **Where does it belong?** | Behind a preprocessing pipeline, usually on a GPU |
| **When should I use it?** | When a human currently looks at an image to make a decision |
