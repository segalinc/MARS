
# MULTIBOX - MOUSE DETECTION

This is an implementation of the Multibox detection system proposed by Szegedy et al. in Scalable High Quality Object
Detection. Currently this repository uses the [Inception-Reset-v2](https://arxiv.org/abs/1602.07261) network as the base network.

paper: [https://arxiv.org/abs/1412.1441](https://arxiv.org/abs/1412.1441)  
data: [http://www.vision.caltech.edu/~segalinc/git_data/develop/](http://www.vision.caltech.edu/~segalinc/git_data/develop/)

![alt text](http://yeephycho.github.io/blog_img/Inception_v4_hires.jpg)

##  PREPARE INPUT FOR MULTIBOX DETECTOR FROM AMT ANNOTATIONS
[https://github.com/gvanhorn38/inception/tree/master/inputs](https://github.com/gvanhorn38/inception/tree/master/inputs)

All files related to the **top view** detection can be found [here](http://www.vision.caltech.edu/~segalinc/git_data/develop/top/).


#### STEPS:  

0. AMT_annotation steps are required to be done before proceeding further
1. Check if saved images are ok: check_img_fromat.sh
2. Convert dictionary of info into tf records in order to make it work with the detector. Set the three sets of data: dict2tfrecords.py
3. step to train, test and evaluation your models are in `cmds_top_all_set_separate.txt`

#### DETAILS: 

The inputs to the detection and pose share a common format. The data need to be stored in an Example protocol buffer with the following fields (not all of them are required):

| Key  | Value |
|------|-------|
|image/id	| string, containing an identifier for this image|
|image/encoded | string, containing JPEG encoded image in RGB colorspace|
|image/height | integer, image height in pixels|
|image/width | integer, image width in pixels|
|image/colorspace | string, specifying the colorspace, always 'RGB' |
|image/channels	| integer, specifying the number of channels, always 3 |
|image/format| string, specifying the format, always 'JPEG'|
|image/filename	| string containing the basename of the image file|
|image/class/label|	integer specifying the index in a classification layer. The label ranges from [0, num_labels), e.g 0-99 if there are 100 classes.|
|image/class/text	| string specifying the human-readable version of the label |
|image/object/bbox/xmin|a float array, the left edge of the bounding boxes; normalized coordinates|
|image/object/bbox/xmax|	a float array, the right edge of the bounding boxes; normalized coordinates|
|image/object/bbox/ymin|	a float array, the top left corner of the bounding boxes; normalized coordinates|
|image/object/bbox/ymax|	a float array, the top edge of the bounding boxes; normalized coordinates.
|image/object/bbox/label|	an integer array, specifying the index in a classification layer. The label ranges from [0, num_labels)|
|image/object/bbox/count|	an integer, the number of bounding boxes|

Note the bounding boxes are stored in normalized coordinates, meaning that the x values have been divided by the width
of the image, and the y values have been divided by the height of the image. This ensures that the object location can
be recovered on any (aspect-perserved) resized version of the original image.  

The `create_tfrecords.py` file has a convience function for generating the tfrecord files. You will need to preprocess
your dataset and get it into a python list of dicts (done in the `A_annotation` folder).
Each dict represents an image and should have the following format:

```python
{
  "filename" : "the full path to the image",
  "id" : "an identifier for this image",
  "class" : {
    "label" : "integer in the range [0, num_classes)",
    "text" : "a human readable string for this class"
  },
  "object" : {
    "bbox" : {
      "xmin" : "an array of float values",
      "xmax" : "an array of float values",
      "ymin" : "an array of float values",
      "ymax" : "an array of float values",
      "label" : "an array of integer values, in the range [0, num_classes)",
      "count" : "an integer, the number of bounding boxes"
    }
  }
}
```
If you want to train a detector, you will need to have at least the following structure:

```python
{
  "filename" : "the full path to the image",
  "id" : "an identifier for this image",
  "object" : {
    "bbox" : {
      "xmin" : "an array of float values",
      "xmax" : "an array of float values",
      "ymin" : "an array of float values",
      "ymax" : "an array of float values",
      "count" : "an integer, the number of bounding boxes"
    }
  }
}
```

Currently, the code assumes that you are working with JPEG images.  
Once you have your dataset preprocessed, you can use a method in `create_tfrecords.py` to create the tfrecords files
as done in `AMT2tfrecords.py`. For example, within a python script or terminal:

```python
train_dataset = None # this should be your array of dicts. Don't forget to separate your training and testing data.

from inputs.create_tfrecords import create
create(
  dataset=train_dataset,
  dataset_name="train",
  output_directory="/home/gvanhorn/Desktop/train_dataset",
  num_shards=10,
  num_threads=5
)
```
## TRAIN AND TEST MULTIBOX DETECTOR

##### DETECTION:

1. Generate priors for the bounding boxes: `prior_generator.py` has both functions to generate priors from hand defined aspect ratios or use the training dataset to cluster the aspect ratios of the bboxes
2. Next we need a configuration file:  save in the dataset folder one file for each type of dateset: `config.yaml.example`
    I set the output as `tf_dataset_detection` folder related to a particular dataset
    Some important settings inlcude:  
    -NUM_BBOXES_PER_CELL: number of aspect ratios you used to generate the priors. The output layers of the model depend on this number  
    -MAX_NUM_BBOXES:to properly pad all the image annotation data in a batch, the maximum number of boxes in a single image must be known  
    -BATCH_SIZE: Depending on your hardware setup, you will need to adjust this parameter so that the network fits in memory  
    -NUM_TRAIN_EXAMPLES: this, along with BATCH_SIZE, is used to compute the number of iterations in an epoch  
    -NUM_TRAIN_ITERATIONS:	This is how many iterations to run before stopping  
    Check out also the other conf parameters
3. `visualize_input_imsize.py` debug your image augmentation setting by visualizing the inputs to the network
4. Train_warmup: Before training the whole network, you need to warmup the detection heads. I call this finetuning.  
5. Train_detection: Therefore the training procedure consists of 2 calls to the `train.py` script. First you finetune. Once the detection heads have warmed up, you can train the whole model. Check the training sometimes checking on the tensorboard w.r.t. the validation set
6. `visualize_val.py`: visualize predicted bboxes and gt. Run `visualize_val_imsize` if you have a validation set, you can visualize the ground thuth boxes and the predicted boxes
7. `detect.py'`: at application time you can run the detect script to generate predicted boxes on new images.
8. `visualize_detect.py`: you can debug the detection setting by using another visualization script. Using `visualize_detect_imsize.py` you can plot the images at original size with the confidence and bounding boxes

##### COCO evaluation:

Note: You can download COCO evaluation tool from [here](https://github.com/pdollar/coco). Remember to run make on PythonAPI in order to get it work 

- `eval.py` : evaluate boxes with coco evaluation metrics exectue eval_prcurve
- to plot precision-recall curve : `prcurve_separate.py`

#### DETAILS:

The input functions to the model require a specific dataset format. You can create the dataset using the utility functions
`create_tfrecords.py` as explained above.

You'll also need to genertate the priors for the bounding boxes. In the `priors.py` file you will find convenience
functions for generating the priors. For example, assuming you are in a python terminal (in the project directory):

```python
import cPickle as pickle
import priors

aspect_ratios = [1, 2, 3, 1./2, 1./3]
p = priors.generate_priors(aspect_ratios, min_scale=0.1, max_scale=0.95, restrict_to_image_bounds=True)
with open('priors.pkl', 'w') as f:
  pickle.dump(p, f)
```

Instead of hand defining the aspect ratios, you can use your training dataset to cluster the aspect ratios of the
bounding boxes. There is a utility function to do this in the `priors.py` file.

```python
aspect_ratios = generate_aspect_ratios(dataset, num_aspect_ratios=11, visualize=False, warp_bboxes=True)
p = generate_priors(aspect_ratios, min_scale=0.1, max_scale=0.95, restrict_to_image_bounds=True)
with open('priors.pkl', 'w') as f:
  pickle.dump(p, f)
```

Next, you'll need to create a configuration file. Checkout the example to see the different settings.  
Some especially important settings include:

**Dataset info**

| Key	 | Value |
|------|-------|
|NUM_BBOXES_PER_CELL|	  To be set to number of aspect ratios you used to generate the priors. The output layers of the model depend on this number|
|MAX_NUM_BBOXES	 |     To properly pad all the image annotation data in a batch, the maximum number of boxes in a single image must be known|
|BATCH_SIZE	|          Depending on your hardware setup, you will need to adjust this parameter so that the network fits in memory. The number of images to process in one iteration|
|NUM_TRAIN_EXAMPLES|	  This, along with BATCH_SIZE, is used to compute the number of iterations in an epoch. It's the number of images in your training tfrecords|
|NUM_TRAIN_ITERATIONS | This is how many iterations to run before stopping|

**Image processing and augmentation**

Deep neural networks are notoriously data hungry. One technique for increasing the amount of data that you can pass through
the network is to augment your training data. Augmentations can be as simple as randomly flipping the images horizontally,
or as complex as extracting crops and perturbing the pixel values. You will typically only want to augment data for the training phase.

|Config Name|	Description|
|-----------|------------|
|INPUT_SIZE	| All images will be resized to [INPUT_SIZE, INPUT_SIZE, 3] prior to passing through the network. You'll want to set this to the same value that the pretrained model used. |
|DO_RANDOM_FLIP_LEFT_RIGHT|	bool,	if true, then each region has a 50% chance of being flipped.|
|DO_COLOR_DISTORTION| float,	value between 0 and 1. 0 means never distort the color, and 1 means always distort the color.|
|COLOR_DISTORT_FAST |	bool, its possible to distort the brightness, saturation, hue and contrast of an image. If true, then slower modifications (hue and contrast) are avoided.|

**Random cropping**

Each region that is extracted from an image can then be randomly cropped. Again, this is a form of data augmentation.
We are trying to make the network robust to changes in the data that do not effect the class label.

You'll definitely want to go through the other configuration parameters, but make sure you have the above parameters set correctly.

Now that you have your dataset in tfrecords, you generated priors, and you set up your configuration file,
you'll be able to train the detection model. First, you can debug your image augmentation setting by visualizing the inputs to the network:

```sh
python visualize_inputs.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/train* \
--config ../tf_dataset_detection/top_all_set_separate/black/config_train.yaml
```

Once you are ready for training, you should download the pretrained **inception-resnet-v2** network and use it as a starting point.
Before training the whole network, you need to warmup the detection heads. I call this finetuning.
A training protocol that typically leads to good performance is to first train only those new layers that get initialized
with random values, leaving the rest of the weights fixed. Once those new layers have "warmed up" then we will train all
of the weights together. However, this "warm up" step is not necessary and we could just train all of the weights together
from the get go. From my experience, this "warm up" step does not always lead to better performance, but I have not seen
it lead to worse performance when compared to a model that was trained without the "warm up" step.
For the warm up phase we won't worry about decaying the learning rate and instead opt to fix it at a relatively high value.
How will we know when the new layers have warmed up? We'll use our validation data and monitor when performance starts to plateau.
Therefore the training procedure consists of 2 calls to the train.py script. First you finetune:

```sh
python train.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/train* \
--priors ../tf_dataset_detection/top_all_set_separate/black/priors.pkl \
--logdir ../tf_dataset_detection/top_all_set_separate/black/finetune \
--config ../tf_dataset_detection/top_all_set_separate/black/config_train.yaml \
--pretrained_model ./inception_resnet_v2_2016_08_30.ckpt \
--fine_tune
```

Once the detection heads have warmed up, you can train the whole model:

```sh
python train.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/train* \
--priors ../tf_dataset_detection/top_all_set_separate/black/priors.pkl \
--logdir ../tf_dataset_detection/top_all_set_separate/black \
--config ../tf_dataset_detection/top_all_set_separate/black/config_train.yaml \
--pretrained_model ../tf_dataset_detection/top_all_set_separate/black/finetune
```

If you have a validation set, you can visualize the ground truth boxes and the predicted boxes:

```sh
python visualize_val.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/val* \
--priors  ../tf_dataset_detection/top_all_set_separate/black/priors.pkl \
--checkpoint_path ../tf_dataset_detection/top_all_set_separate/black/model.ckpt-300000 \
--config ../tf_dataset_detection/top_all_set_separate/black/config_train.yaml
```

At "application time" you can run the detect script to generate predicted boxes on new images.
You can debug your detection setting by using another visualization script:

```sh
python detect.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/test* \
--priors ../tf_dataset_detection/top_all_set_separate/black/priors.pkl \
--checkpoint_path ../top_all_set_separate/black/data_10k_top/model.ckpt-300000 \
--save_dir ../tf_dataset_detection/top_all_set_separate/black/ \
--config ../tf_dataset_detection/top_all_set_separate/black/config_detect.yaml \
--max_iterations 0

python visualize_detect.py \
--tfrecords ../tf_dataset_detection/top_all_set_separate/black/test* \
--priors ../tf_dataset_detection/top_all_set_separate/black/priors.pkl \
--checkpoint_path ../tf_dataset_detection/top_all_set_separate/black/model.ckpt-300000 \
--config ../tf_dataset_detection/top_all_set_separate/black/config_detect.yaml
```

**Export & Compress**

To export a model for easy use on a mobile device you can use:

```sh
python export.py \
--checkpoint_path ../tf_dataset_detection/top_all_set_separate/black \
--export_dir ../tf_dataset_detection/top_all_set_separate/black \
--export_version 1 \
--config ../tf_dataset_detection/config_export.yaml
```
The input node is called images and the output node is called Predictions.

If you are going to use the model with TensorFlow Serving then you can use the following:
```sh
python export.py \
--checkpoint_path ../tf_dataset_detection/top_all_set_separate/black \
--export_dir ../tf_dataset_detection/top_all_set_separate/black \
--export_version 1 \
--config ../tf_dataset_detection/top_all_set_separate/black/config_export.yaml \
--serving
```


##                    FILES INCLUDED IN THIS FOLDER

|Name |                                Description|
|-----|--------------------------------------------|
|cmds_front_allset_separate|           bash command to run multibox detection, step by step on front view videos
|cmds_top_allset_separate|             bash command to run multibox detection, step by step on top view videos
|config.py|                            parse the config.yaml|
|create_tfrecords.py|                  utility to convert dictionary into tf format|
|detect.py          |                  inference on test data|
|detect_imsize.py     |                inference on test data returning results on image size scale|
|export.py      |                      export a model, loading in the moving averages and saving those|
|inputs.py          |                  input pipeline for training the detector|
|inputs_moreinfo.py  |                 input pipeline for training the detector where I added path filenames and actions|
|loss.py        |                      loss function|
|model.py      |                       network model|
|priors.py          |                  utility to generate aspect rations and priors bboxes|
|prior_generator.py |                  utility to generate the prior based on priors.py|
|train.py        |                     finetune and trains the network|
|visualize_detect.py    |              visualize inference result|
|visualize_detect_imsize.py   |        visualize inference result on original image|
|visualize_detect_separate_json.py |   visualize inference separate mice from json files|
|visualize_inputs_imsize.py  |         visualization of input on the original image size|
|visualize_inputs.py |                 visualization of input to check that everything is set up correctly on the resized images with bboxes|
|visualize_val.py   |                  visualize prediction and gt on images|
|visualize_val_imsize.py  |            visualize prediction and gt on original image size|

`/evaluation`: pycocotools, coco, build folders for COCO evaluation

|Name |                                Description|
|--------------------|----------------------|
|check_results_id.py|                 check that ids and results are ok|
|eval.py               |               use the COCO evaluation pipeline to evaluate performances|
|eval_inputs.py       |                input pipeline for training the evaluation|
|prcurve_separate.py  |           compute pr curves from eval.py saved pickle cocoEval file|


#### ARCHITECTURE

  * Basic structure of Inception-ResNet-v2 (layers, dimensions):
    * `Image -> Stem -> 5x Module A -> Reduction-A -> 10x Module B -> Reduction B -> 5x Module C -> AveragePooling -> Droput 20% -> Linear, Softmax`
    * `299x299x3 -> 35x35x256 -> 35x35x256 -> 17x17x896 -> 17x17x896 -> 8x8x1792 -> 8x8x1792 -> 1792 -> 1792 -> 1000`
  * Modules A, B, C are very similar.
  * They contain 2 (B, C) or 3 (A) branches.
  * Each branch starts with a 1x1 convolution on the input.
  * All branches merge into one 1x1 convolution (which is then added to the original input, as usually in residual architectures).
  * Module A uses 3x3 convolutions, B 7x1 and 1x7, C 3x1 and 1x3.
  * The reduction modules also contain multiple branches. One has max pooling (3x3 stride 2), the other branches end in convolutions with stride 2.
  
  *From top to bottom: Module A, Module B, Module C, Reduction Module A.*
  
  




