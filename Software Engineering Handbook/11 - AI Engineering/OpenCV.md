# OpenCV

> **In one line —** the standard toolbox for everything you do to an image before and after a model sees it, plus the classical algorithms that still beat neural networks under controlled conditions.

| | |
|---|---|
| **Full name** | Open Source Computer Vision Library |
| **Category** | Library |
| **Architectural Layer** | Application |
| **Written in** | C++, with bindings for Python, Java and others |
| **Related notes** | [Computer Vision](Computer%20Vision.md) · [YOLO](YOLO.md) · [OCR](OCR.md) · [Deep Learning](Deep%20Learning.md) |

---

## 1. Short Definition

*What is it?*

OpenCV is a library of image and video processing functions: reading, transforming, filtering, detecting features, and running classical computer vision algorithms. It also has a DNN module for running trained models.

---

## 2. Purpose

*What is its main purpose?*

To handle everything around the model — loading, resizing, colour conversion, drawing results — and to solve directly the problems that do not need a model at all.

---

## 3. Problem

*What engineering problem does it solve?*

Every vision system needs the same operations: decode an image, resize it, change colour space, crop, rotate, threshold, find contours, draw a box. Reimplementing those correctly and efficiently would be a project in itself.

---

## 4. Architecture Position

```text
Camera / file / stream
    ↓
OPENCV — decode, preprocess          ← you are here
    ↓
Model (PyTorch / ONNX / OpenCV DNN)
    ↓
OPENCV — postprocess, annotate, encode
    ↓
Output
```

---

## 5. The BGR trap

> [!CAUTION]
> **OpenCV loads images as BGR, not RGB.** This is a decision from the early 2000s that cannot now be changed, and it is the single most common bug in computer vision code.

```python
img = cv2.imread("photo.jpg")          # BGR
img = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)   # ← almost always required
```

The consequence is not an error. The model runs, returns confident predictions, and is quietly wrong — because it was trained on RGB and is being fed inverted colour channels.

---

## 6. What it is genuinely good at

```text
✓ Reading and writing images and video, in every format
✓ Resizing, cropping, rotating, warping
✓ Colour space conversion
✓ Thresholding and morphology
✓ Contour detection — finding shapes
✓ Edge detection (Canny), corners, blobs
✓ Template matching
✓ Camera calibration and perspective correction
✓ Motion detection through frame differencing
✓ Drawing boxes, masks and text onto output
```

> [!TIP]
> **Document scanning is a good example of classical vision winning outright.** Find edges → find the largest quadrilateral contour → perspective-warp it flat. No model, no training data, no GPU, deterministic, milliseconds. A neural network would be slower and worse.

---

## 7. Classical or model?

```text
Are the conditions CONTROLLED?
  fixed camera · consistent lighting · known object shape
    ↓ YES                             ↓ NO
Classical OpenCV                 Train or fine-tune a model
  fast, free, deterministic         handles variation
  brittle if conditions change      needs data and compute
```

Real systems usually combine both: OpenCV finds and straightens the region of interest, then a model interprets its contents.

---

## 8. Video capture

```python
cap = cv2.VideoCapture(0)
while True:
    ret, frame = cap.read()
    if not ret: break
    # process
cap.release()
```

> [!CAUTION]
> **`VideoCapture` buffers frames.** If your processing is slower than the stream, you are reading increasingly stale frames from a growing buffer — the display lags further behind reality the longer it runs. For live systems, read in a dedicated thread and always process the most recent frame, discarding the backlog.

---

## 9. Real World Example

- **Preprocessing for every deep learning vision pipeline** — this is its most common role.
- **Barcode and QR detection**, document straightening, licence plate region extraction.
- **Motion-triggered recording** in surveillance, avoiding constant inference.
- **Industrial measurement** against a fixed jig with controlled lighting.

---

## 10. Communication and Dependencies

- **NumPy** — OpenCV images *are* NumPy arrays in Python, which is why the ecosystem interoperates so easily
- **PyTorch / ONNX** — for the model step
- **FFmpeg** underneath, for video codecs
- **`opencv-python-headless`** for servers — the standard build pulls in GUI dependencies you do not want in a container

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use OpenCV in every computer vision project — if only for image loading and preprocessing. Use its classical algorithms wherever the environment is controlled.

> [!CAUTION]
> Do not use classical techniques for problems with real-world variation: recognising objects in arbitrary photographs, reading handwriting, or anything where lighting and angle change. Threshold-and-contour pipelines tuned to one lighting condition break the moment someone opens a blind.

---

## 12. Advantages and Disadvantages

**Advantages**
- Extremely fast — optimised C++ with SIMD
- Comprehensive, covering essentially every image operation
- Deterministic and testable
- No GPU required
- Mature, stable and universally available

**Disadvantages**
- **BGR by default** — a permanent source of bugs
- Classical algorithms are brittle under changing conditions
- The API is C++-shaped, and not always Pythonic
- Poor error messages
- The full package pulls in heavy GUI dependencies unless you use headless

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Speed** | Microseconds to milliseconds per operation |
| **CPU** | Efficient; many operations are SIMD-optimised |
| **GPU** | Optional CUDA module; rarely necessary |
| **Memory** | Images are large — a 4K frame is ~25 MB uncompressed |
| **Video** | Decoding is often more expensive than the processing |

> [!TIP]
> **Resize early.** Running operations on a 4K frame when 640×480 would answer the question wastes most of the time in the pipeline.

---

## 14. Security Considerations

> [!CAUTION]
> **Image decoding is parsing untrusted binary input in C++.** Image libraries have a long history of memory-safety vulnerabilities exploited through malformed files. An endpoint that accepts user uploads and decodes them is a genuine attack surface.

- **Validate file size and dimensions before decoding** — a "decompression bomb" is a small file that expands to gigabytes
- **Keep OpenCV updated** — it has published CVEs
- **Process uploads in an isolated worker**, not in the web process
- **Strip EXIF metadata** — it frequently contains GPS coordinates
- **Never pass user input into file paths** for `imread`/`imwrite`

---

## 15. Mental Model

> [!NOTE]
> **OpenCV is the darkroom, not the photographer.**
>
> It crops, straightens, adjusts contrast and prints — everything around the picture. The judgement about what is *in* the picture is the model's job. And the darkroom is far cheaper than the photographer, so do as much there as you can.

---

## 16. Mini Architecture Diagram

```text
Input (file · camera · stream)
    ↓
cv2.imread / VideoCapture      → BGR array
    ↓
cvtColor → RGB                 ← do not forget this
resize · crop · normalise
    ↓
Model
    ↓
cv2.rectangle · putText        → annotate results
    ↓
imwrite / imencode / display
```

---

## 17. Complete Request Flow

Document scanning, end to end, with no model at all:

```text
Photo of a document uploaded
    ↓
Size and dimensions validated before decoding
    ↓
cv2.imread → BGR
    ↓
Convert to greyscale
    ↓
Gaussian blur → reduce noise
    ↓
Canny edge detection
    ↓
findContours → largest 4-sided contour = the page
    ↓
getPerspectiveTransform + warpPerspective → flattened page
    ↓
Adaptive threshold → clean black-and-white scan
    ↓
Passed to OCR
    ↓
Total: a few milliseconds, deterministic, no training data
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> OpenCV handles everything around the model and solves controlled-condition problems outright — and it loads images as BGR, which will silently break your model if you forget.

---

## 19. Common Mistakes

- **Forgetting BGR → RGB** conversion before a model
- **Different resize interpolation** between training and serving
- **Processing full-resolution frames** unnecessarily
- **Not draining the video capture buffer**, so live feeds lag
- **Classical pipelines tuned to one lighting condition**
- **Installing the full package in a container** instead of headless
- **Decoding untrusted images in the web process**

---

## 20. Open Source Technologies

- **OpenCV** (`opencv-python-headless` for servers)
- **Pillow**, **scikit-image** — alternatives for simpler image handling
- **NumPy** — the underlying array type
- **FFmpeg** — video decoding
- **Albumentations** — augmentation, built on OpenCV

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Load an image with OpenCV and display it without converting BGR to RGB. Note what happens.
- [ ] Build the document-scanning pipeline from section 17 with no model.
- [ ] Time one operation at full resolution and at 640×480.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Input → OpenCV preprocessing → model → OpenCV postprocessing → output
```

## 2. Request Flow

```text
Input       an image or video frame
    ↓
Processing  decoded, colour-converted, resized, transformed
    ↓
Output      a model-ready tensor, or a finished result with no model at all
```

## 3. Real-World Usage

**Document scanning in mobile applications** — edge detection plus perspective warping — is pure classical OpenCV. It runs on a phone in milliseconds and needs no training data, which is why it has not been replaced by a neural network.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The standard library for image and video processing |
| **Why does it exist?** | So every project does not reimplement image operations |
| **Where does it belong?** | Around the model, on both sides |
| **When should I use it?** | Always — and for the whole problem when conditions are controlled |
