# Getting started

## Prebuilt models

DeepForest has a prebuilt model trained on data from 24 sites from the [National Ecological Observation Network](https://www.neonscience.org/field-sites/field-sites-map). The prebuilt model uses a semi-supervised approach in which millions of moderate quality annotations are generated using a LiDAR unsupervised tree detection algorithm, followed by hand-annotations of RGB imagery from select sites.

![](../www/semi-supervised.png)
For more details on the modeling approach see [citations](landing.html#citation).

The prebuilt model was trained on 0.1m data in window sizes of 400px. As an initial test of performance, we recommend staying around this ground area for each prediction window. For example, if you had 2m satellite data, and wanted to predict on a similar ground area: 2m/0.1m = 20 and 400px/20=20px, so you should predict_images of 20px to use the prebuilt model.

## Sample data

DeepForest comes with a small set of sample data to help run the docs examples. Since users may install in a variety of manners, and it is impossible to know the relative location of the files, the helper function ```get_data``` is used. This function looks to where DeepForest is installed, and finds the deepforest/data/ directory.

```
YELL_xml = get_data("2019_YELL_2_541000_4977000_image_crop.xml")
```

## Prediction

DeepForest allows convenient prediction of new data based on the prebuilt model or a [custom trained](getting_started.html#Training) model. There are three ways to format data for prediction.

### Predict a single image

For single images, ```predict_image``` can read an image from memory or file and return predicted tree bounding boxes.

```{python}
from deepforest import deepforest
from deepforest import get_data

test_model = deepforest.deepforest()
test_model.use_release()

#Predict test image and return boxes
#Find path to test image. While it lives in deepforest/data, its best to use the function if installed as a python module
image_path = get_data("OSBS_029.tif")
boxes = test_model.predict_image(image_path=image_path, show=False, return_plot = False)

boxes.head()
```

```
xmin        ymin        xmax        ymax     score label
0  222.136353  211.271133  253.000061  245.222580  0.790797  Tree
1   52.070221   73.605804   82.522354  111.510605  0.781306  Tree
2   96.324028  117.811966  123.224060  145.982407  0.778245  Tree
3  336.983826  347.946747  375.369019  396.250580  0.677282  Tree
4  247.689362   48.813339  279.102570   88.318176  0.675362  Tree
```

### Predict a tile

Large tiles covering wide geographic extents cannot fit into memory during prediction and would yield poor results due to the density of bounding boxes. Often provided as geospatial .tif files, remote sensing data is best suited for the ```predict_tile``` function, which splits the tile into overlapping windows, perform prediction on each of the windows, and then reassembles the resulting annotations.

```
from deepforest import deepforest
from deepforest import get_data

test_model = deepforest.deepforest()
test_model.use_release()

raster_path = get_data("OSBS_029.tif")
#Window size of 300px with an overlap of 50% among windows
predicted_raster = test_model.predict_tile(raster_path, return_plot = True, patch_size=300,patch_overlap=0.5)
```

### Predict a set of annotations

During evaluation of ground truth data, it is useful to have a way to predict a set of images and combine them into a single data frame. The ```predict_generator``` method allows a user to point towards a file of annotations and returns the predictions for all images.

Consider a headerless annotations.csv file in the following format

```
image_path, xmin, ymin, xmax, ymax, label
```
with each bounding box on a seperate row. The image path is relative to the local of the annotations file.

```
from deepforest import deepforest
from deepforest import get_data

test_model = deepforest.deepforest()
test_model.use_release()

annotations_file = get_data("testfile_deepforest.csv")

#Window size of 300px with an overlap of 50% among windows
boxes = test_model.predict_generator(annotations=annotations_file)
```

For more information on data files, see below.

## Training

The prebuilt models will always be improved by adding data from the target area. In our work, we have found that even one hour's worth of carefully chosen hand-annotation can yield enormous improvements in accuracy and precision. We envision that for the majority of scientific applications atleast some finetuning of the prebuilt model will be worthwhile.

Consider an annotations.csv file in the following format

```
image_path, xmin, ymin, xmax, ymax, label
```

testfile_deepforest.csv

```
OSBS_029.jpg,256,99,288,140,Tree
OSBS_029.jpg,166,253,225,304,Tree
OSBS_029.jpg,365,2,400,27,Tree
OSBS_029.jpg,312,13,349,47,Tree
OSBS_029.jpg,365,21,400,70,Tree
OSBS_029.jpg,278,1,312,37,Tree
OSBS_029.jpg,364,204,400,246,Tree
OSBS_029.jpg,90,117,121,145,Tree
OSBS_029.jpg,115,109,150,152,Tree
OSBS_029.jpg,161,155,199,191,Tree
```

and a classes.csv file in the same directory

```
Tree,0
```

```{python}
from deepforest import deepforest
from deepforest import get_data

test_model = deepforest.deepforest()

# Example run with short training
test_model.config["epochs"] = 1
test_model.config["save-snapshot"] = False
test_model.config["steps"] = 1

annotations_file = get_data("testfile_deepforest.csv")

test_model.train(annotations=annotations_file, input_type="fit_generator")
```

```{python}
No model initialized, either train or load an existing retinanet model
There are 1 unique labels: ['Tree']
Disabling snapshot saving

Training retinanet with the following args ['--backbone', 'resnet50', '--image-min-side', '800', '--multi-gpu', '1', '--epochs', '1', '--steps', '1', '--batch-size', '1', '--tensorboard-dir', 'None', '--workers', '1', '--max-queue-size', '10', '--freeze-layers', '0', '--score-threshold', '0.05', '--save-path', 'snapshots/', '--snapshot-path', 'snapshots/', '--no-snapshots', 'csv', 'data/testfile_deepforest.csv', 'data/classes.csv']

Creating model, this may take a second...

... [omitting model summary]

Epoch 1/1

1/1 [==============================] - 11s 11s/step - loss: 4.0183 - regression_loss: 2.8889 - classification_loss: 1.1294
```

## Evaluation

Independent analysis of whether a model can generalize from training data to new areas is critical for creating a robust workflow. We stress that evaluation data must be different from training data, as neural networks have millions of parameters and can easily memorize thousands of samples. Therefore, while it would be rather easy to tune the model to get extremely high scores on the training data, it would fail when exposed to new images.

DeepForest uses the keras-retinanet ```evaluate``` method to score images. This consists of an annotations.csv file in the following format

```
image_path, xmin, ymin, xmax, ymax, label
```

```{python}
from deepforest import deepforest
from deepforest import get_data

test_model = deepforest.deepforest()
test_model.use_release()

annotations_file = get_data("testfile_deepforest.csv")
mAP = test_model.evaluate_generator(annotations=annotations_file)
print("Mean Average Precision is: {:.3f}".format(mAP))
```

```{python}
Running network: 100% (1 of 1) |#########| Elapsed Time: 0:00:02 Time:  0:00:02
Parsing annotations: N/A% (0 of 1) |     | Elapsed Time: 0:00:00 ETA:  --:--:--
Parsing annotations: 100% (1 of 1) |#####| Elapsed Time: 0:00:00 Time:  0:00:00
60 instances of class Tree with average precision: 0.3687
mAP using the weighted average of precisions among classes: 0.3687
mAP: 0.3687
```
