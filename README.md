# Thermal Face

Fast face detection in thermal images

> TODO: Write summary (referencing [Fever](https://github.com/maxbbraun/fever)).

## Inference

Face detection is optimized for edge devices such as [Coral](https://coral.ai/docs/accelerator/get-started/).

Inference from Python can be done manually using the [TF Lite API](https://www.tensorflow.org/lite/guide/python) or more easily using the [Edge TPU API](https://coral.ai/docs/edgetpu/api-intro/):

```bash
pip3 install Pillow
sudo apt-get install python3-edgetpu
```

```python
from edgetpu.detection.engine import DetectionEngine
from PIL import Image

# One-time initialization:
face_detector = DetectionEngine('thermal_face_automl_edge_l_edgetpu.tflite')

# Per-image detection:
image = Image.open('image.png').convert('RGB')
faces = face_detector.detect_with_image(image,
                                        threshold=0.1,
                                        keep_aspect_ratio=True,
                                        relative_coord=False,
                                        top_k=10)
for face in faces:
  print(face.bounding_box)  # np.array([[x1, y1], [x2, y2]], dtype=float64)
```

> TODO: Profile image conversions. Reduce allocations with more buffers?

> TODO: Add note about performance results. Mention Coral maximum operating frequency.

## Training

> TODO: Write intro.

#### 1. Create the dataset

> TODO: Add non-thermal face database (biasing toward thermal in validation and test sets).
> WIDER_train.zip https://drive.google.com/uc?export=download&id=0B6eKvaijfFUDQUUwd21EckhUbWs
> WIDER_val.zip https://drive.google.com/uc?export=download&id=0B6eKvaijfFUDd3dIRmpvSk8tLUk
> WIDER_test.zip https://drive.google.com/uc?export=download&id=0B6eKvaijfFUDbW4tdGpaYjgzZkU
> http://mmlab.ie.cuhk.edu.hk/projects/WIDERFace/support/bbx_annotation/wider_face_split.zip

Download the thermal images from the [Tufts Face Database](http://tdface.ece.tufts.edu) and upload them to [Cloud Storage](https://cloud.google.com/storage/docs):

```bash
cd training

LOCATION="us-central1"
TDFACE_DIR="tufts-face-database"
TDFACE_BUCKET="gs://$TDFACE_DIR"
MODEL_BUCKET="gs://thermal-face"

for i in $(seq 1 4)
do
  curl -O http://tdface.ece.tufts.edu/downloads/TD_IR_A/TD_IR_A_Set$i.zip
  curl -O http://tdface.ece.tufts.edu/downloads/TD_IR_E/TD_IR_E_Set$i.zip
done

mkdir $TDFACE_DIR
for f in TD_IR_*.zip
do
  unzip $f -d $TDFACE_DIR/$(basename $f .zip)
done

gsutil mb -l $LOCATION $TDFACE_BUCKET
gsutil -m rsync -r $TDFACE_DIR $TDFACE_BUCKET
gsutil mb -l $LOCATION $MODEL_BUCKET
```

Create a dataset spec in the [AutoML format](https://cloud.google.com/vision/automl/object-detection/docs/csv-format) using the matching [bounding box annotations](https://github.com/maxbbraun/tdface-annotations):

```bash
curl -O https://raw.githubusercontent.com/maxbbraun/tdface-annotations/master/bounding-boxes.csv
TDFACE_ANNOTATIONS="bounding-boxes.csv"
AUTOML_SPEC="automl.csv"

python3 -m venv venv
. venv/bin/activate
pip3 install -r requirements.txt

python automl_convert.py \
  --tdface_dir=$TDFACE_DIR \
  --tdface_bucket=$TDFACE_BUCKET \
  --tdface_annotations=$TDFACE_ANNOTATIONS \
  --tdface_automl=$AUTOML_SPEC

gsutil cp $AUTOML_SPEC $MODEL_BUCKET
```

#### 2. Train the model

```bash
MODEL_NAME="thermal_face_automl_edge_l"
```

Using [Cloud AutoML Vision](https://cloud.google.com/vision/automl/object-detection/docs/edge-quickstart):

 - Model objective: **Object detection**
 - CSV file on Cloud Storage: **`$MODEL_BUCKET/$AUTOML_SPEC`**
 - Model name: **`$MODEL_NAME`**
 - Model type: **Edge**
 - Optimize for: **Higher accuracy**
 - Node budget: **24 node hours**
 - Use model: **TF Lite**
 - Export to Cloud Storage: **`$MODEL_BUCKET/`**

#### 3. Compile the model

Use [Docker](https://docs.docker.com) to compile the model for [Edge TPU](https://coral.ai/products/):

```bash
MODEL_FILE="$MODEL_NAME.tflite"
TPU_MODEL_FILE="${MODEL_FILE%.*}_edgetpu.${MODEL_FILE##*.}"

gsutil cp $MODEL_BUCKET/**/*$MODEL_NAME*/model.tflite $MODEL_FILE

docker build -t edgetpu_compiler --build-arg MODEL_FILE=$MODEL_FILE .
docker run edgetpu_compiler
docker cp $(docker ps -alq):/$TPU_MODEL_FILE .

mv $MODEL_FILE ../
mv $TPU_MODEL_FILE ../
```

> TODO: Debug issues with edgetpu-compiled model.
