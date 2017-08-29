This project shows how one can use machine learning to enable the recognition of complex gestures in an iOS app. See the article here: [A new approach to touch-based mobile interaction](https://medium.com/@mitochrome/a-new-approach-to-touch-based-mobile-interaction-ba47b14400b0)

The sample data and demo app support 13 gestures which the latter distinguishes with high accuracy: check marks, x marks, ascending diagonals, "scribbles" (rapid side-to-side motion while moving either up or down), circles, U shapes, hearts, plus signs, question marks, capital A, capital B, happy faces and sad faces.

File layout:
* apps/
  * **GestureInput**: An iOS app for inputting examples of gestures and maintaining a set of them. Runs on iOS 10.
  * **GestureRecognizer**: An iOS app that uses the model trained by gesturelearner to classify gestures drawn by the user in real time. Specifically, it contains a Core ML .mlmodel file generated by [`gesturelearner/save_mlmodel.py`](https://github.com/mitochrome/complex-gestures-demo/blob/master/gesturelearner/save_mlmodel.py). Since Core ML is new with iOS 11, this app only runs there.
* **gesturelearner**: Scripts that manipulate the set of gestures produced by GestureInput and use TensorFlow to train a model that will recognize them
* sample_data: A sample data set from GestureInput with 1300 examples of 13 different gestures (100 each). Also includes a trained TensorFlow model.

## Setup

Since GestureRecognizer requires iOS 11, you'll need to have Xcode 9 beta set up and a device running iOS 11 beta.

Set up GestureInput and GestureRecognizer:
```
# Make sure Xcode 9 is selected.
# sudo xcode-select -s /Applications/Xcode-beta.app/Contents/Developer

cd apps/GestureInput
carthage bootstrap --platform iOS
cd ../GestureRecognizer
carthage bootstrap --platform iOS
```

If you just want to try out GestureRecognizer, you don't need to train a new model yourself. Otherwise, set up gesturelearner with virtualenv:
```
cd gesturelearner
virtualenv -p $(which python3) venv
pip install -r requirements.txt
```

## Transferring data to and from your device

GestureInput saves the data set in two files in the Documents folder of its application container:

* **dataset.dataset**: The primary storage, which stores the full time sequence of touch positions for each example. This file is only used by GestureInput.
* **dataset.trainingset**: This file is generated by the "Rasterize" button. It converts all the gesture examples into images, storing them and their labels in this file. This is the file that gesturelearner uses for training.

Follow [these instructions](https://stackoverflow.com/a/28161494) to download the container and get dataset.trainingset.

If you want to add to the sample data, you'll need to transfer it to the device. Download the application container, put the sample dataset.trainingset file in the Documents folder and replace the container.

## Training

Start by activating gesturelearner's virtual environment with `source venv/bin/activate`.

Typical use of gesturelearner would be:
```
# Split the rasterized data set into an 85% training set and 15% test set (the holdout method).
# Replace `/path/to/gesturelearner` with the actual path.
python /path/to/gesturelearner/filter.py --test-fraction=0.15 data.trainingset

# Convert the generated files to TensorFlow data files.
python /path/to/gesturelearner/convert_to_tfrecords.py data_filtered.trainingset
python /path/to/gesturelearner/convert_to_tfrecords.py data_filtered_test.trainingset

# Train the neural network.
python /path/to/gesturelearner/train.py --test-file=data_filtered_test.tfrecords data_filtered.tfrecords

# Save a Core ML .mlmodel file.
python /path/to/gesturelearner/save_mlmodel.py model.ckpt
```

The generated model.mlmodel file can be added to Xcode 11 which will automatically generate a Swift type for using the model. The model used by GestureRecognizer is at `GestureRecognizer/Source/Resources/GestureModel.mlmodel`.

## Adding new gesture classes

GestureInput, GestureRecognizer and gesturelearner all get the list of classes from the `Label` enum in `touches.proto`, which is currently duplicated at [`apps/Common/Protobuf/touches.proto`](https://github.com/mitochrome/complex-gestures-demo/blob/master/apps/Common/Protobuf/touches.proto) and [`gesturelearner/protobuf/touches.proto`](https://github.com/mitochrome/complex-gestures-demo/blob/master/gesturelearner/protobuf/touches.proto).

If you add a gesture class to `touches.proto`, you'll need to:
- Make sure you keep the list of label values in ascending order.
- If you want to use the sample data provided in this repository, make sure you preserve the enum values of the existing gesture classes.
- Run the protobuf `protoc` utility to regenerate `touches.pb.swift` and `touches_pb2.py`. For `touches.pb.swift` you'll need to install [`swift-protobuf`](https://github.com/apple/swift-protobuf) and follow the instructions at that repository. For `touches_pb2.py`, see [these instructions](https://developers.google.com/protocol-buffers/docs/reference/python-generated).
- Add your new class to `Touches_Label.all` in [`apps/Common/Label+Extensions.swift`](https://github.com/mitochrome/complex-gestures-demo/blob/master/apps/Common/Label%2BExtensions.swift) so that the order of classes there matches their order in `touches.proto`.
- In `Label+Extensions.swift`, give your new class a name to be displayed in the iOS apps.
