# PCB Automated Optical Inspection

**Final Year Project · Electrical and Electronic Engineering · University of Peradeniya**

A camera-based Automated Optical Inspection (AOI) system for detecting and categorizing small defects on bare Printed Circuit Boards (PCBs). The project combines a Raspberry Pi capture node, image preprocessing, a custom PCBNet-v5 detector and a separate defect classifier.

The intended deployment uses an **edge-to-cloud architecture**: the Raspberry Pi captures and preprocesses images, while a cloud service performs model inference and returns the inspection results.

> **Status: Under active development.** Model training, camera-image testing and preprocessing experiments are being refined. Raspberry Pi integration and cloud deployment are planned integration work. Inspection speed and accuracy will be reported from measured evaluations; deployment targets are not presented as achieved results.

## Project objectives

- Develop an affordable inspection setup using a Raspberry Pi and camera.
- Locate small PCB defects while retaining sufficient image detail.
- Classify detected candidates as open circuit, short circuit, mouse bite, pinhole or other.
- Adapt models trained on DeepPCB to photographs from the project hardware.
- Move compute-intensive inference from the capture device to a cloud backend.

## Inspection workflow

1. **Capture and preprocess:** acquire an RGB board photograph, isolate the board, orient it into landscape and produce a binary image with black copper traces and white substrate.
2. **Detect:** apply PCBNet-v5 to obtain a defect-candidate probability map. Convert accepted regions into bounding boxes using the selected testing pipeline.
3. **Classify:** extract clean crops from the input image using the candidate coordinates, retain surrounding trace context and predict each crop's defect category.
4. **Display:** show bounding boxes, candidate counts and classified crops. Low-confidence predictions can be reported as uncertain using a separately configured classifier threshold.

For the planned networked system, `client.py` will send a PNG image in an HTTPS POST request. A FastAPI service will run inference and return structured results, such as coordinates, categories and scores.

**The thresholds have different roles:** the preprocessing threshold assigns copper versus substrate; the detector threshold selects candidate regions; the classifier threshold controls whether a category prediction is accepted. Their values are not interchangeable.

## PCBNet-v5 detector

The custom detector combines:

- A five-stage backbone using multiscale residual blocks.
- Local and dilated depthwise convolution branches with lightweight channel attention.
- A Feature Pyramid Network (FPN) for combining features at different scales.
- A deterministic 17-channel shallow feature extractor using gradients, Laplacians, Gaussian responses and local statistics.
- A learned 32-channel shallow embedding and a 128-channel FPN embedding.
- A segmentation head that fuses the 160 channels and produces a single-channel sigmoid output.

DeepPCB bounding boxes are converted into filled rectangular training masks. Consequently, this training setup learns **defect regions**, not exact pixel-level outlines of damaged copper.

The classifier is a separate model. See the classifier notebook for the architecture, preprocessing and class order associated with each checkpoint.

## Defect categories

| Category | Meaning |
|---|---|
| Open circuit | A break that interrupts a copper connection. |
| Short circuit | An unwanted copper bridge between conductors. |
| Mouse bite | A notch of missing copper at a trace or pad boundary. |
| Pinhole | A small void inside a copper region; it does not have to be associated with a component pin. |
| Other | The additional defect examples assigned to the trained Other class. |

`Other` is a learned category. An uncertain prediction is a separate outcome, rather than automatically belonging to Other. Intentional drill holes are normal board features and must be distinguished from actual defects during annotation and evaluation.

## Image preprocessing

The current colour-histogram notebook includes:

- Automatic board-boundary detection and cropping.
- Optional perspective straightening and landscape orientation.
- Manual selection of at least three pure copper and three pure substrate sample regions.
- Comparison of Lab a/b, HSV H/S and full Lab material histograms.
- Binary conversion, colour-score maps and uncertainty overlays.

This is currently an interactive preprocessing experiment. Reusing calibration or automating material sampling under fixed capture conditions is needed before unattended deployment.

Temporary board-location masks may use morphological closing. They are discarded before trace conversion. The final material-conversion stage does not apply morphological hole filling. Perspective straightening resamples RGB pixels once; it can be disabled to preserve original pixels using a rectangular crop. Colour errors can still remove tiny gaps or holes, so outputs must be compared with the source photograph.

## Dataset and evaluation

The project uses [DeepPCB](https://github.com/tangsanli5201/DeepPCB) together with annotated camera images for domain adaptation.

The baseline DeepPCB split is:

| Split | Images | Purpose |
|---|---:|---|
| Training | 800 | Model fitting |
| Validation | 200 | Checkpoint selection and threshold tuning |
| Official test | 500 | Final benchmark evaluation |

The baseline detector uses 256 × 256 inputs, batch size 8, AdamW and a combined binary cross-entropy and Dice loss. The original configuration allows up to 50 epochs with early stopping. The selected notebook records the actual settings for each experiment.

Camera adaptation currently uses annotations from two physical boards, with a third board reserved for testing. All photographs and crops from a physical board must remain in the same split. Three boards provide limited evidence of generalization; repeated photographs do not count as new board designs.

Evaluation should report detector precision/recall, classifier per-class precision/recall and confusion matrices, together with end-to-end results on held-out boards. False positives around intentional holes and missed small defects remain areas of investigation. Thresholds should be selected using validation data and fixed before final testing.

## Repository folders

| Folder | Contents |
|---|---|
| `notebooks/` | Training, testing, preprocessing and fine-tuning notebooks. |
| `models/` | Detector and classifier definitions, plus checkpoint instructions. |
| `preprocessing/` | Reusable board-cropping and material-conversion scripts. |
| `edge/` | Raspberry Pi capture, lighting and upload code as integration progresses. |
| `server/` | FastAPI backend and container configuration as integration progresses. |
| `examples/` | Selected camera inputs, binary images and inspection outputs. |
| `data/` | Dataset preparation, annotation examples and split information. |
| `docs/` | Architecture, hardware setup, experiments and measured results. |

Some folders may initially contain documentation while code is being organized.

## Getting started

1. Open the relevant notebook from `notebooks/` in Google Colab or Kaggle.
2. Follow its setup instructions and attach the required dataset, checkpoint or camera image.
3. Update the configuration paths before running the cells in order.
4. For detector inference, use the model definition and preprocessing that match the saved checkpoint.
5. For classification, load the classifier checkpoint and its recorded class order, then classify the clean candidate crops.
6. Compare outputs against annotations or reviewed examples before reporting performance.

Training notebooks require a suitable GPU for practical runtimes. The colour-histogram preprocessing notebook runs on CPU. Python dependencies vary by notebook and include PyTorch, torchvision, OpenCV, NumPy, SciPy, Pillow and Matplotlib.

Dataset and checkpoint download instructions will be maintained in the corresponding folders. The complete datasets and large weights are not intended to be committed directly with the source code.

## Hardware and planned deployment

| Component | Role |
|---|---|
| Raspberry Pi 5 | Intended capture and preprocessing node running Raspberry Pi OS, a Linux-based operating system. |
| Raspberry Pi Camera Module 2 | Board-image capture. |
| Controlled lighting and custom enclosure | More consistent image acquisition. |
| Python and OpenCV | Capture-side image processing. |
| FastAPI and Uvicorn | Planned inference API. |
| Docker | Planned backend packaging. |
| Google Cloud Run | Target hosting platform, subject to deployment and performance testing. |

## Team

This project is developed by three undergraduate team members.

| Member | Registration number | GitHub | Contribution |
|---|---|---|---|
| D. M. M. L. Dissanayake | E/21/109 | [movindu23](https://github.com/movindu23) | Add agreed responsibilities and completed work. |
| J.J.M.Y.A.B. JAYASUNDARA | E/21/197 | Adithya62 | Add agreed responsibilities and completed work. |
| B.S.M.L.U. BANDARA | E/21/047 | Add GitHub username | Add agreed responsibilities and completed work. |

## Collaboration

Use individual GitHub accounts and descriptive commits. Develop changes on a branch and open a pull request for review before merging into `main`. Record experiment settings, checkpoint identity, dataset splits and measured results so team members can reproduce the work.

## Acknowledgements

- [DeepPCB](https://github.com/tangsanli5201/DeepPCB) for the PCB defect dataset.
- [PyTorch](https://pytorch.org/), [OpenCV](https://opencv.org/) and the open-source tools used by the project.
- Pretrained classifier models and any reused implementations should be credited in the corresponding notebooks.

Third-party datasets, pretrained weights and source code retain their respective licences. Licensing for the team's original code will be specified when agreed by the team.
