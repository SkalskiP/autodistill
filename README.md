<div align="center">
  <p>
    <a align="center" href="" target="_blank">
      <img
        width="850"
        src="https://media.roboflow.com/open-source/autodistill/autodistill-banner.png"
      >
    </a>
  </p>
</div>

## Autodistill

Autodistill is an ecosystem of Python packages for distilling large foundation models into smaller, faster models suitable for deployment. Using `autodistill`, you can go from images to inference on a custom model running at the edge in realtime with little to no labeling or human intervention in between.

<div align="center">
  <p>
    <a align="center" href="" target="_blank">
      <img
        width="850"
        src="https://media.roboflow.com/open-source/autodistill/steps.jpg"
      >
    </a>
  </p>
</div>

Autodistill is designed as a simple framework for plugging in new base and target models, experimenting with prompts, supporting human-in-the-loop model-assisted labeling, evaluating results, and deploying to the edge. Currently, `autodistill` supports vision tasks like object detection and instance segmentation, but in the future it can be expanded to support language (and other) models.

## Basic Concepts

To use `autodistill`, you input unlabeled data into a Base Model which uses an Ontology to label a Dataset which is used to train a Target Model that outputs a Distilled Model fine-tuned to perform a specific Task.

<div align="center">
  <p>
    <a align="center" href="" target="_blank">
      <img
        width="850"
        src="https://media.roboflow.com/open-source/autodistill/overview.jpg"
      >
    </a>
  </p>
</div>

That's a lot of vocabulary; let's break it down bit by bit:

* **Task** - A Task defines what a Target Model will predict. The Task for each component (Base Model, Ontology, and Target Model) of an `autodistill` pipeline must match for them to be compatible with each other. Object Detection and Instance Segmentation are currently supported through the `detection` task. `classification` support will be added soon.
* **Base Model** - A Base Model is a large foundation model that knows a lot about a lot. Base models are often multimodal and can perform many tasks. They're large, slow, and expensive. Examples of Base Models are GroundedSAM and GPT-4's upcoming multimodal variant. We use a Base Model (along with unlabeled input data and an Ontology) to create a Dataset.
* **Ontology** - an Ontology defines how your Base Model is prompted, what your Dataset will describe, and what your Target Model will predict. A simple Ontology is the `CaptionOntology` which prompts a Base Model with text captions and maps them to class names. Other Ontologies may, for instance, use a CLIP vector or example images instead of a text caption.
* **Dataset** - a Dataset is a set of auto-labeled data that can be used to train a Target Model. It is the output generated by a Base Model.
* **Target Model** - a Target Model is a supervised model that consumes a Dataset and outputs a distilled model that is ready for deployment. Target Models are usually small, fast, and fine-tuned to perform a specific task very well (but they don't generalize well beyond the information described in their Dataset). Examples of Target Models are YOLOv8 and DETR.
* **Distilled Model** - a Distilled Model is the final output of the `autodistill` process; it's a set of weights fine-tuned for your task that can be deployed to get predictions.

## Installation

Autodistill is modular. You'll need to install the base `autodistill` package (which defines the interfaces for the above concepts) along with Base Model and Target Model plugins (which implement specific models).

By packaging these separately as plugins, dependency and licensing incompatibilities are minimized and new models can be implemented and maintained by anyone.

Example: 
```bash
pip install autodistill autodistill-grounded-sam autodistill-yolov8
```

<details close>
<summary>Install from source</summary>

You can also clone the project from GitHub for local development:

```bash
git clone https://github.com/autodistill/autodistill
cd autodistill
pip install -e .
```
</details>

Additional Base and Target models are enumerated below.

## 🚀 Quickstart

See the [Quickstart.ipynb](Quickstart.ipynb) notebook for a quick introduction to `autodistill`. Below, we have condensed key parts of the notebook for a quick introduction to `autodistill`.

### Install Packages

For this example, we'll show how to distill [GroundedSAM](https://github.com/IDEA-Research/Grounded-Segment-Anything) into a small [YOLOv8](https://github.com/ultralytics/ultralytics) model using [autodistill-grounded-sam](https://github.com/autodistill/autodistill-grounded-sam) and [autodistill-yolov8](https://github.com/autodistill/autodistill-yolov8).

```
pip install autodistill autodistill-grounded-sam autodistill-yolov8
```

### Distill a Model

```python
from autodistill_grounded_sam import GroundedSAM
from autodistill.detection import CaptionOntology
from autodistill_yolov8 import YOLOv8

# define an ontology to map class names to our GroundingDINO prompt
# the ontology dictionary has the format {caption: class}
# where caption is the prompt sent to the base model, and class is the label that will
# be saved for that caption in the generated annotations
base_model = GroundedSAM(ontology=CaptionOntology({"shipping container": "container"}))

# label all images in a folder called `context_images`
base_model.label(
  input_folder="./images",
  output_folder="./dataset"
)

target_model = YOLOv8("yolov8n.pt")
target_model.train("./dataset/data.yaml", epochs=200)

# run inference on the new model
pred = target_model.predict("./dataset/valid/your-image.jpg", confidence=0.5)
print(pred)

# optional: upload your model to Roboflow for deployment
from roboflow import Roboflow

rf = Roboflow(api_key="API_KEY")
project = rf.workspace().project("PROJECT_ID")
project.version(DATASET_VERSION).deploy(model_type="yolov8", model_path=f"./runs/detect/train/")
```

### Annotate a Single Image

To plot the annotations for a single image using `autodistill`, you can use the code below. This code is helpful to visualize the annotations generated by your base model (i.e. GroundedSAM) and the results from your target model (i.e. YOLOv8).

```python
import supervision as sv
import cv2

img_path = "./images/your-image.jpeg"

image = cv2.imread(img_path)

detections = base_model.predict(img_path)
# annotate image with detections
box_annotator = sv.BoxAnnotator()

labels = [
    f"{base_model.ontology.classes()[class_id]} {confidence:0.2f}"
    for _, _, confidence, class_id, _ in detections
]

annotated_frame = box_annotator.annotate(
    scene=image.copy(), detections=detections, labels=labels
)

sv.plot_image(annotated_frame, (16, 16))
```

## 📍 Roadmap

Our goal is for `autodistill` to support using all foundation models as Base Models and most SOTA supervised models as Target Models. We focused on object detection and segmentation
tasks first but plan to launch classification support soon! In the future, we hope `autodistill` will also be used for models beyond computer vision.

<div align="center">
  <p>
    <a align="center" href="" target="_blank">
      <img
        width="850"
        src="https://media.roboflow.com/open-source/autodistill/connections.jpg"
      >
    </a>
  </p>
</div>

✅ - complete (click to go to repo)
🚧 - work in progress

### object detection

| base / target       | YOLOv5 | YOLOv7 | [YOLOv8](https://github.com/autodistill/autodistill-yolov8) | RT-DETR |
|:-------------------:|:------:|:------:|:------:|:-------:|
| GroundingDINO       |        |        |        |         |
| [Grounded SAM](https://github.com/autodistill/autodistill-grounded-sam)        |        |        | ✅     |         |
| DETIC               |        |        |        |         |
| OWL-ViT             |        |        |        |         |

### instance segmentation

| base / target       | YOLOv5 | YOLOv7 | [YOLOv8](https://github.com/autodistill/autodistill-yolov8) | RT-DETR |
|:-------------------:|:------:|:------:|:------:|:-------:|
| [Grounded SAM](https://github.com/autodistill/autodistill-grounded-sam)        |        |        | ✅     |         |

### classification

| base / target | YOLOv8 | YOLOv5 | ViT |
|:-------------:|:------:|:------:|:----|
| CLIP          |        |        |     |

## 🏆 Contributing

We love your input! Please see our [contributing guide](CONTRIBUTING.md) to get started. Thank you 🙏 to all our contributors!

## License

The `autodistill` package is licensed under an [Apache 2.0](LICENSE.md). Each Base or Target model plugin may use its own license corresponding with the license of its underlying model. Please refer to the license in each plugin repo for more information.