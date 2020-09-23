# Face Detection

Models that are able to detect faces.

| Model Name | Complexity (GFLOPs) | Size (Mp) | AP @ [IoU=0.50:0.95] (%) | AP for faces > 64x64 (%) | WiderFace Easy (%) | WiderFace Medium (%) | WiderFace Hard (%) | Links | GPU_NUM |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| face-detection-0200 | 0.82 | 1.83 | 16.0 | 86.743 | 82.917 | 76.198 | 41.443 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0200.pth), [model template](./face-detection-0200/template.yaml) | 2 |
| face-detection-0202 | 1.84 | 1.83 | 20.3 | 91.938 | 89.382 | 83.919 | 50.189 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0202.pth), [model template](./face-detection-0202/template.yaml) | 2 |
| face-detection-0204 | 2.52 | 1.83 | 21.4 | 92.888 | 90.453 | 85.448 | 52.091 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0204.pth), [model template](./face-detection-0204/template.yaml) | 4 |
| face-detection-0205 | 2.94 | 2.02 | 21.6 | 93.566 | 92.032 | 86.717 | 54.055 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0205.pth), [model template](./face-detection-0205/template.yaml) | 4 |
| face-detection-0206 | 340.06 | 63.79 | 34.2 | 94.274 | 94.281 | 93.207 | 84.439 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0206.pth), [model template](./face-detection-0206/template.yaml) | 8 |
| face-detection-0207 | 1.04 | 0.81 | 17.2 | 88.17 | 84.406 | 76.748 | 43.452 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/face-detection-0207.pth), [model template](./face-detection-0207/template.yaml) | 1 | 

## Training pipeline

### 0. Change a directory in your terminal to object_detection.

```bash
cd <training_extensions>/pytorch_toolkit/object_detection
```

### 1. Select a model template file and instantiate it in some directory.

```bash
export MODEL_TEMPLATE=./model_templates/face-detection/face-detection-0200/template.yaml
export WORK_DIR=/tmp/face-detection-0200
python tools/instantiate_template.py ${MODEL_TEMPLATE} ${WORK_DIR}
```

### 2. Collect dataset

Download the [WIDER Face](http://shuoyang1213.me/WIDERFACE/) and unpack it to the `${DATA_DIR}` folder.

```bash
export DATA_DIR=${WORK_DIR}/data
```

### 3. Prepare annotation

Convert downloaded and extracted annotation to MSCOCO format with `face` as the only one class.

* Training annotation

   ```bash
   export TRAIN_ANN_FILE=${DATA_DIR}/train.json
   export TRAIN_IMG_ROOT=${DATA_DIR}
   python ./model_templates/face-detection/tools/wider_to_coco.py \
      ${DATA_DIR}/wider_face_split/wider_face_train_bbx_gt.txt \
      ${DATA_DIR}/WIDER_train/images/ \
      ${TRAIN_ANN_FILE}
   ```

* Validation annotation

   ```bash
   export VAL_ANN_FILE=${DATA_DIR}/val.json
   export VAL_IMG_ROOT=${DATA_DIR}
   python ./model_templates/face-detection/tools/wider_to_coco.py \
      ${DATA_DIR}/wider_face_split/wider_face_val_bbx_gt.txt \
      ${DATA_DIR}/WIDER_val/images/ \
      ${VAL_ANN_FILE}
   ```

### 4. Change current directory to directory where the model template has been instantiated.

```bash
cd ${WORK_DIR}
```

### 5. Training and Fine-tuning

Try both following variants and select the best one:

   * **Training** from scratch or pre-trained weights. Only if you have a lot of data, let's say tens of thousands or even more images. This variant assumes long training process starting from big values of learning rate and eventually decreasing it according to a training schedule.
   * **Fine-tuning** from pre-trained weights. If the dataset is not big enough, then the model tends to overfit quickly, forgetting about the data that was used for pre-training and reducing the generalization ability of the final model. Hence, small starting learning rate and short training schedule are recommended.

   * If you would like to start **training** from pre-trained weights use `--load-weights` pararmeter. Parameters such as `--epochs`, `--batch-size` and `--gpu-num` can be omitted, default values will be loaded from `${MODEL_TEMPLATE}`. Please be aware of default values for these parameters in particular `${MODEL_TEMPLATE}`.

      ```bash
      export EPOCHS_NUM=70
      export GPUS_NUM=1
      export BATCH_SIZE=32

      python train.py \
         --load-weights ${WORK_DIR}/snapshot.pth \
         --train-ann-files ${TRAIN_ANN_FILE} \
         --train-img-roots ${TRAIN_IMG_ROOT} \
         --val-ann-files ${VAL_ANN_FILE} \
         --val-img-roots ${VAL_IMG_ROOT} \
         --save-checkpoints-to ${WORK_DIR}/outputs \
         --epochs ${EPOCHS_NUM} \
         --batch-size ${BATCH_SIZE} \
         --gpu-num ${GPUS_NUM}
      ```

   * If you would like to start **fine-tuning** from pre-trained weights use `--resume-from` pararmeter and value of `--epochs` have to exceeds value stored inside `${MODEL_TEMPLATE}` file, otherwise training will be ended immideately. Parameters such as `--batch-size` and `--gpu-num` can be omitted, default values will be loaded from `${MODEL_TEMPLATE}`.  Please be aware of default values for these parameters in particular `${MODEL_TEMPLATE}`.

      ```bash
      export EPOCHS_NUM=75
      export GPUS_NUM=1
      export BATCH_SIZE=32

      python train.py \
         --resume-from ${WORK_DIR}/snapshot.pth \
         --train-ann-files ${TRAIN_ANN_FILE} \
         --train-img-roots ${TRAIN_IMG_ROOT} \
         --val-ann-files ${VAL_ANN_FILE} \
         --val-img-roots ${VAL_IMG_ROOT} \
         --save-checkpoints-to ${WORK_DIR}/outputs \
         --epochs ${EPOCHS_NUM} \
         --batch-size ${BATCH_SIZE} \
         --gpu-num ${GPUS_NUM}
      ```

### 6. Evaluation

Evaluation procedure allows us to get quality metrics values and complexity numbers such as number of parameters and FLOPs.

To compute MS-COCO metrics and save computed values to `${WORK_DIR}/metrics.yaml` run:

```bash
python eval.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-img-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml
```

You can also save images with predicted bounding boxes using `--save-output-images-to` parameter.

```bash
python eval.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-img-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml \
   --save-output-images-to ${WORK_DIR/}/output_images
```

If you have WiderFace dataset downloaded you also can specify `--wider-dir` parameter where `WIDER_val.zip` file is stored in order to compute official WiderFace metrics.

```bash
python eval.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-img-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml \
   --wider-dir ${DATA_DIR}
```

### 6. Export PyTorch\* model to the OpenVINO™ format

To convert PyTorch\* model to the OpenVINO™ IR format run the `export.py` script:

```bash
python export.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --save-model-to ${WORK_DIR}/export
```

This produces model `model.xml` and weights `model.bin` in single-precision floating-point format
(FP32). The obtained model expects **normalized image** in planar BGR format.

For SSD networks an alternative OpenVINO™ representation is done automatically to `${WORK_DIR}/export/alt_ssd_export` folder.
SSD model exported in such way will produce a bit different results (non-significant in most cases),
but it also might be faster than the default one. As a rule SSD models in [Open Model Zoo](https://github.com/opencv/open_model_zoo/) are exported using this option.

### 7. Validation of IR

Instead of passing `snapshot.pth` you need to pass path to `model.bin` (or `model.xml`).

```bash
python eval.py \
   --load-weights ${WORK_DIR}/export/model.bin \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-img-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml
```