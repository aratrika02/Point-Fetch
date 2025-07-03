# Point & Fetch
## Overview
Point & Fetch, is a vision-language-enabled assistive system which aims to to help individuals with physical or speech impairments interact wittheir environment through intuitive, non-verbal gestures. The system allows users to simply pointat an object, upon which it interprets the intent and detects the object to be fetched. By leveraging recent advances in object detection, human pose estimation, and large language models (LLMs), our modular framework integrates gesture recognition with contextual understanding to determine the user’s needs.

## Design and Development

![Design Flow](https://github.com/aratrika02/Point-Fetch/blob/main/flow.jpg)

Our design comprises three key modules: MediaPipe, YOLOv8, and Deepseek. We use MediaPipe and YOLO to calculate the geometrically most likely object to be fetched and then leverage the reasoning powers of DeepSeek to bring contextual understanding into the fold, to determine whether the object is reasonable to fetch or not.
### 1. MediaPipe
We use MediaPipe (specifically, MediaPipe Hands), a lightweight, high-fidelity hand and finger tracker for gesture (pointing) detection. 
We use MediaPipe Hands to track and detect a single hand, first in a static image, then in a video. From this, we get normalized 3D landmark coordinates for key points such as the wrist (landmark 0) and index-finger tip (landmark 8). After loading the image/video, we convert it from OpenCV’s BGR format to RGB, following which the Hands module detects the presence of a hand. Subsequently, we extract the 3D coordinates of the wrist and index fingertip in the normalized space and convert them into 2D pixel space, following which the pointing vector is calculated as:

Pointing Vector = Index fingertip coordinates - Wrist coordinates

### 2. YOLOv8
We use Ultralytics’ YOLOv8 from the You Only Look Once family for real time object detection. In our Python pipeline, we use the yolov8 small model to
process static images first and then videos. At each frame (or image), we call model(image) to run inference, internally resizing and normalizing the input, then extract each detection’s pixel-space bounding box (xyxy), confidence score, and class ID. We filter out the “person” class so the user’s own hand, nor any other human person isn’t mistaken for a fetchable object, compute each remaining box’s center point (object vector), and store its label and centroid.

### 3. Cosine Similarity Score
 We normalize the object and pointing vectors to unit length and calculate the cosine similarity between them to find which object is most aligned with the
direction the user is pointing towards.

### 4. DeepSeek
We integrate DeepSeek-R1, a firstgeneration reasoning LLM trained via large-scale reinforcement learning (RL), to make the final fetch/no-fetch decision
based on geometric matching.
- **Understanding the true intent of the user** : Geometric pointing is not enough to infer the underlying intent of the user because sometimes the user’s pointing direction may be askew or ambiguous, resulting in the wrong object being detected for fetching. While carrying out our own experiments, it so happened that the MediaPipe+YOLO pipeline picked up the ‘couch’ that ought to be fetched, when in reality, a bottle was being pointed at. However, the direction of pointing was a bit off, leading to the couch being detected as the object to be fetched. We integrate the LLM component as a failsafe to mitigate scenarios precisely like this, where the geometrically detected object may be an unreasonable object to be fetched, given environmental context.
- **Taking into account user preferences** : Building on the theme of ambiguity in the direction of pointing and subsequently, the computed cosine similarity score, let us consider an example of a user pointing at a book placed on a cluttered desk. It will become difficult for the geometric part of our model to accurately identify what the user wants. This is where we bring in the LLM. From its memory of user preferences and past user choices, it can reason that since the user has previously asked for the book, it must be the book that ought to be fetched from the desk.
- **Ensuring user safety** : It may be that the detected object is too sharp or too hot and fetching and handing it to the user may cause harm. In such cases, the LLM (Deepseek) can reason to not fetch the object and in urgent cases, alert the human caregiver.
