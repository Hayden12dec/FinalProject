# AI-Based Smart Examination Proctoring System
## Comprehensive System Architecture, Workflow & Technology Specification Guide

---

## 1. Executive Overview

The **AI-Based Smart Examination Proctoring System** is an automated academic integrity and online evaluation platform designed to monitor candidate behavior during remote assessments. It utilizes real-time computer vision, deep learning, and ambient acoustic telemetry to detect unauthorized activities (unauthorized devices, multiple individuals, gaze deviations, candidate absence, and impersonation) without invasive surveillance or unfair automatic disqualification.

The platform couples real-time telemetry with an **Explainable Suspicion Scoring Engine**, debounced evidence capture, and human-in-the-loop audit reporting for academic administrators and chief proctors.

---

## 2. 4-Tier System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│ 1. CLIENT TIER (React 18 / Vite SPA)                                   │
│    • WebRTC Camera Stream (720p/1080p Video)                           │
│    • Web Audio API (In-Browser RMS Acoustic Energy Analyzer)           │
│    • Frame Throttler & Base64 Encoder (~1 sample per 800ms-1200ms)     │
│    • Canvas HUD Dynamic Overlay (Real-time 2D Bounding Boxes & Badges) │
│    • Timed MCQ Assessment Engine & Tab-Switch / Blur Listeners         │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │ HTTPS / REST (Base64 Frames + RMS)
┌────────────────────────────────────▼───────────────────────────────────┐
│ 2. APPLICATION & ROUTING TIER (FastAPI Server)                         │
│    • REST API Router Layer (/api/proctoring, /api/attempts, etc.)      │
│    • JWT Bearer Authentication & Role-Based Authorization              │
│    • Proctoring Service Coordinator Layer                              │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │
┌────────────────────────────────────▼───────────────────────────────────┐
│ 3. AI & COMPUTER VISION PIPELINE                                       │
│    • Multi-Stage Face & Presence: MediaPipe Short/Full-Range & FaceMesh │
│    • 3D Head Pose & Gaze Estimation: OpenCV solvePnP (Yaw, Pitch, Roll)│
│    • Prohibited Object Detection: YOLOv8n (Phones, Laptops, Books)     │
│    • Biometric Identity & Impersonation: SFace 128-D Deep CNN + Ratios │
│    • Acoustic Energy Analyzer: RMS Threshold Speech/Noise Detection    │
│    • Suspicion Scoring Engine: Debounce Cooldown (5s) & Points Matrix  │
└────────────────────────────────────┬───────────────────────────────────┘
                                     │
┌────────────────────────────────────▼───────────────────────────────────┐
│ 4. PERSISTENT STORAGE TIER (Hybrid Storage Engine)                     │
│    • Primary Database: MongoDB via PyMongo                             │
│    • Fault-Tolerant Fallback: Thread-Safe Local JSON Store (data_store)│
│    • Photographic Evidence Store: Timestamped Annotated JPEGs (evidence)│
└────────────────────────────────────────────────────────────────────────┘
```

### Architectural Component Roles
1. **Client Tier (Frontend)**: Runs entirely inside modern desktop web browsers. It accesses the webcam and microphone via HTML5 WebRTC, calculates acoustic volume locally to protect privacy, throttles frame captures to conserve network bandwidth, and overlays bounding boxes onto the candidate's video feed.
2. **Application Tier (Backend)**: Built with asynchronous FastAPI and Uvicorn. Decodes client payloads, authenticates requests via JWT, delegates computer vision tasks, and routes telemetry.
3. **AI Pipeline**: Orchestrates specialized machine learning models (MediaPipe, YOLOv8, SFace, solvePnP) in parallel or cascaded sequences to identify exam violations within <100ms per frame.
4. **Persistent Storage Tier**: A hybrid abstraction layer that writes to MongoDB if active, or seamlessly falls back to a thread-safe local JSON document store, ensuring no exam session is interrupted by database downtime.

---

## 3. End-to-End System Workflow

```
[Phase 1: Registration] ──► [Phase 2: System Check] ──► [Phase 3: Live Proctoring Loop]
                                                                     │
                         [Phase 5: Proctor Audit] ◄── [Phase 4: Submission & Scoring]
```

### Phase 1: Candidate Enrollment & Biometric Baseline
1. Candidate registers an account with their academic credentials.
2. The user captures a clear, front-facing webcam image.
3. The biometric verification module generates a 128-dimensional facial embedding and geometric landmark baseline, persisting it as `face_reference` in the user's profile.

### Phase 2: 5-Point Automated Pre-Exam System Check
Before the exam unlocks, the candidate completes an automated hardware and environment audit:
1. **Camera Feed Check**: Confirms browser media permissions and active stream.
2. **Microphone Audio Check**: Verifies microphone input and measures ambient background noise.
3. **Lighting Quality Check**: Calculates image brightness and contrast thresholds.
4. **Single-Candidate Presence**: Confirms exactly one face is positioned in the camera frame.
5. **Biometric Identity Match**: Compares the live face against the enrolled `face_reference`. If biometric similarity is below ~60%, examination access is locked to prevent impersonation.

### Phase 3: Live Exam & Real-Time AI Proctoring Loop
During the active test, a recurring loop executes every **800ms – 1200ms**:
1. **Client Frame Sampling**: The browser captures a video frame, calculates acoustic RMS energy, and transmits a payload to `POST /api/proctoring/frame`.
2. **AI Anomaly Detection Suite**:
   - **Absence / Multi-Person**: Detects if the candidate leaves (>8 seconds) or if secondary individuals enter the room.
   - **Gaze & Head Pose**: Calculates 3D Euler angles (Yaw, Pitch, Roll) using OpenCV `solvePnP` to flag looking away.
   - **Prohibited Devices**: YOLOv8n checks for mobile phones, tablets, laptops, and books with temporal multi-frame confirmation.
   - **Periodic Biometric Verification**: Re-verifies facial identity every 12 frames to prevent candidate substitution.
   - **Acoustic Disturbance**: Flags talking or loud ambient noise spikes.
   - **Browser Tab Integrity**: Client-side window listeners flag when the student leaves or minimizes the exam tab (`TAB_SWITCH_DETECTED`).
3. **Debounced Scoring & Evidence Persistence**:
   - If an anomaly is verified and its 5-second cooldown timer has elapsed, the system saves an annotated image snapshot (`/evidence/*.jpg`) and logs an event.
   - Points are added to the cumulative suspicion score.
4. **HUD Telemetry Update**: Real-time status tags, risk levels, and visual bounding boxes are sent back to the candidate's HUD and the proctor's surveillance room.

### Phase 4: Exam Submission & Grading
1. The student submits answers or the timer expires.
2. The exam engine calculates the academic score.
3. The scoring engine evaluates the final suspicion score to classify the attempt:
   - **Low Risk (0 – 29 Points)**: Normal exam conditions.
   - **Medium Risk (30 – 59 Points)**: Minor warnings; flagged for report review.
   - **High Risk (60+ Points)**: Severe infractions; flagged for immediate human investigation.

### Phase 5: Proctor Audit & Report Review
1. Administrators access the **Admin Audit Report View**.
2. Displays candidate metadata, test scores, risk classifications, chronological anomaly event timelines, and an interactive gallery of timestamped evidence snapshots.

---

## 4. Deep-Dive: "IMPERSONATION DETECTED" Alert

### What It Means
`IMPERSONATION_DETECTED` is an automated security violation triggered when the person sitting in front of the camera fails biometric verification against the enrolled profile photo (`face_reference`).

### Underlying Pipeline
1. **Biometric Extraction**: OpenCV SFace (`FaceRecognizerSF`) extracts a 128-dimensional deep feature vector from the live face, alongside 42 skeletal bone-anchored facial landmarks from MediaPipe FaceMesh.
2. **Similarity Comparison**: The engine computes cosine similarity and anthropometric morphometric ratios against the registered reference image.
3. **Threshold Check**: If similarity falls below the required threshold (default ~55%–62%), the system triggers `IMPERSONATION_DETECTED`.
4. **Impact**:
   - Status tag updates to **`🚨 IMPERSONATION DETECTED`**.
   - Adds **+40 suspicion points** to the exam attempt.
   - Captures and saves an annotated evidence JPEG with similarity percentages.
   - If detected during the pre-exam system check, the exam remains locked.

### Typical Causes
- **Different Person / Wrong Account**: Someone other than the registered student is in front of the camera.
- **Outdated / Low-Quality Enrollment Photo**: The baseline photo was captured in poor lighting, is blurry, or had an invalid placeholder.
- **Harsh Lighting / Shadows**: Extreme backlight or shadows obscuring facial landmark extraction.
- **Face Occlusion**: Hands covering the face, masks, or tinted eyewear.

---

## 5. Technology Stack Specification

### Frontend Technologies (Client-Side)
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **React 18** | UI Framework | Powers the Single Page Application, exam interface, admin dashboard, and live telemetry HUD. |
| **Vite** | Build Tool & Server | High-performance build tool with Hot Module Replacement for rapid development. |
| **HTML5 WebRTC** | Media Streaming | Direct browser access to webcam and microphone streams without plugins. |
| **Web Audio API** | Audio Processing | In-browser calculation of Root-Mean-Square (RMS) amplitude for privacy-preserving noise detection. |
| **HTML5 Canvas** | Graphics & HUD | Real-time 2D drawing of face tracking boxes, device markers, and direction badges over video. |
| **Chart.js & react-chartjs-2** | Data Visualization | Renders suspicion timelines, score distribution charts, and risk metrics in audit reports. |
| **Lucide React** | UI Components | Lightweight, modern icon set for status badges, buttons, and navigation elements. |
| **Canvas Confetti** | Animation | Confetti animation triggered upon successful exam completion and submission. |

### AI, Deep Learning & Computer Vision
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **OpenCV (`opencv-python`)** | Computer Vision | Image decoding, color conversion, CLAHE equalization, drawing annotated evidence, and 3D geometry. |
| **MediaPipe** (Google) | Face Tracking | Multi-stage face detection and 468-point 3D FaceMesh landmark extraction. |
| **YOLOv8n (`ultralytics`)** | Deep Neural Network | Real-time object detection for prohibited items (mobile phones, laptops, books). |
| **OpenCV `solvePnP`** | 3D Pose Estimation | Estimates Yaw, Pitch, and Roll head orientation angles to detect looking away. |
| **SFace (`FaceRecognizerSF`)** | Biometric Recognition | 128-D deep neural network facial embedding model for identity verification and impersonation detection. |
| **NumPy** | Numerical Computing | Vector math, Euclidean distances, cosine similarity, and array slicing for image matrices. |
| **Pillow (PIL)** | Image Processing | Image format conversions, thumbnail handling, and snapshot persistence. |

### Backend Technologies (Server-Side)
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **FastAPI** | Web Framework | Asynchronous Python REST API handling frame telemetry, exam delivery, and admin reports. |
| **Uvicorn** | ASGI Web Server | Asynchronous server powering high-concurrency API execution. |
| **Pydantic (v2)** | Data Validation | Strict schema validation and data parsing for API requests and entity models. |
| **`python-multipart`** | File Handling | Streaming parser for candidate profile photo uploads. |
| **`python-dotenv`** | Configuration | Environment configuration loader for secrets, database URLs, and ports. |

### Security & Authentication
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **JWT (`PyJWT`)** | Authentication | Issues and validates digitally signed JSON Web Tokens for secure, stateless sessions. |
| **Bcrypt & Passlib** | Cryptography | One-way password hashing using 12 salt rounds before storage. |

### Database & Storage
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **MongoDB & PyMongo** | NoSQL Database | Primary document database storing users, exams, attempts, proctoring events, and reports. |
| **Resilient Local JSON Store** | Fallback Storage | Thread-safe file-based JSON store in `data_store/` allowing full offline operation without MongoDB. |
| **Disk Evidence Store** | Static Storage | Directory (`evidence/`) preserving timestamped JPEG snapshots with bounding boxes. |

### Testing & Quality Assurance
| Technology | Category | Role in the System |
| :--- | :--- | :--- |
| **Pytest & `pytest-asyncio`** | Test Framework | Automated unit testing and integration testing for asynchronous endpoints and AI modules. |
| **HTTPX & Requests** | HTTP Clients | Programmatic HTTP clients used during automated end-to-end API test suites. |

---

## 6. Anomaly Scoring Matrix & Rules

| Anomaly Event Type | Point Penalty | Risk Category | Trigger Condition |
| :--- | :---: | :---: | :--- |
| `MOBILE_PHONE_DETECTED` | **+50 pts** | High Risk | YOLOv8 detects phone with $\ge 0.75$ confidence or multi-frame persistence. |
| `IMPERSONATION_DETECTED` | **+40 pts** | High Risk | Biometric cosine similarity against registered reference falls below threshold. |
| `MULTIPLE_PERSONS_DETECTED`| **+40 pts** | High Risk | More than 1 face or person detected in the camera frame. |
| `STUDENT_ABSENT` | **+30 pts** | High Risk | Zero faces detected for $>8$ consecutive seconds. |
| `AUDIO_ACTIVITY_DETECTED` | **+20 pts** | Medium Risk | Ambient acoustic RMS amplitude $> 0.03$ continuously. |
| `FACE_NOT_DETECTED` | **+20 pts** | Medium Risk | Face temporarily obscured or positioned outside frame bounds. |
| `TAB_SWITCH_DETECTED` | **+15 pts** | Medium Risk | Browser window loses focus or user switches tabs. |
| `SUSPICIOUS_HEAD_MOVEMENT` | **+10 pts** | Low/Med Risk | $\|Yaw\| > 20^\circ$ (looking left/right) or $\|Pitch\| > 16^\circ$ (looking up/down). |

> **Debounce Rule:** Points and evidence snapshots for the same anomaly type are logged only once every **5 seconds** to prevent score inflation during continuous motion.

---

## 7. Prompts for Generating Visuals (ChatGPT / DALL-E)

### Prompt A: System Architecture Diagram
> "Create a high-resolution, professional technical system architecture diagram for an **AI-Powered Online Exam Proctoring System**.
>
> **Aesthetic & Style:** Modern enterprise cloud architecture infographic, dark-mode sleek aesthetic (`#0c101d` background), semi-transparent glassmorphic cards, glowing cyan, cobalt blue, and purple neon accent borders, clean tech icons, and glowing directional data arrows.
>
> **Diagram Structure (4 Stacked Layers):**
> 1. **Layer 1: Frontend Client Tier (React 18 / Vite)**: WebRTC Camera Stream, Web Audio API (RMS Energy Processor), Frame Throttler (1 frame/sec Base64), Canvas HUD Overlay, Exam Engine & Tab-Blur Listener.
> 2. **Layer 2: Application & Gateway Tier (FastAPI)**: REST API Endpoints, JWT Security & Auth, Proctoring Service Coordinator Layer.
> 3. **Layer 3: AI & Computer Vision Intelligence Suite**: MediaPipe Face & Presence Detector, OpenCV solvePnP 3D Head Pose (Yaw/Pitch/Roll), YOLOv8n Object Detector (Phones/Laptops/Books), SFace Deep CNN Biometric Verifier, Audio Acoustic Processor, and Scoring Engine with 5-Second Cooldown Debouncer.
> 4. **Layer 4: Persistent Data & Evidence Tier**: Hybrid Database (MongoDB / Local JSON Store) and Annotated Evidence Snapshots Directory (`/evidence/*.jpg`).
>
> **Connectors:** Show labeled glowing arrows moving downward for telemetry (Frames, Audio RMS) and upward for proctoring feedback (Risk Scores, Alerts, HUD BBoxes)."

### Prompt B: End-to-End Candidate Workflow Infographic
> "Create a sleek, modern step-by-step process flowchart infographic illustrating the complete **End-to-End Candidate Lifecycle & AI Monitoring Workflow** for an online exam proctoring platform.
>
> **Aesthetic & Style:** Clean isometric roadmap/pipeline style with a dark tech theme, glowing gradient connectors, numbered milestone circular nodes (1 to 5), vibrant status badges, and clear legible typography.
>
> **Workflow Steps:**
> - **Step 1: Biometric Enrollment & Baseline Registration**: Student registers and captures baseline front-facing photo via webcam; system stores reference facial embedding.
> - **Step 2: Automated 5-Point System Pre-Check**: Candidate validates Camera, Microphone RMS, Ambient Lighting, Single Face Presence, and Biometric Identity Verification.
> - **Step 3: Active Exam & Real-Time AI Proctoring Loop**: Continuous recurring loop during the exam; AI monitors for phones, extra persons, head turns, student absence, and tab-switching.
> - **Step 4: Suspicion Scoring & Debounced Evidence Capture**: Assigns points (+10 to +50 pts), applies 5-second cooldown debounce, and persists timestamped JPEG evidence with highlighted bounding boxes.
> - **Step 5: Exam Finalization & Chief Proctor Audit**: Candidate submits test; system calculates grade and classifies risk tier (Low, Medium, High); Admin inspects comprehensive audit report with chronological timeline and evidence gallery."
