Audio Super Resolution Using Neural Networks
============================================

This repository implements the audio super-resolution model proposed in:

```
V. Kuleshov, Z. Enam, and S. Ermon. Audio Super Resolution Using Neural Networks. ICLR 2017 (Workshop track)
V. Kuleshov, Z. Enam, P. W. Koh, and S. Ermon. Deep Convolutional Time Series Translation, ArXiv 2017
```

## Installation

### Requirements

The model is implemented in Tensorflow and Keras and uses several additional libraries. Our configuration was:

* `tensorflow==0.12.1`
* `keras==1.2.1`
* `numpy==1.12.0`
* `scipy==0.18.1`
* `librosa==0.4.3`
* `h5py==2.6.0`
* `matplotlib==1.5.1`

### Setup

To install this package, simply clone the git repo:

```
git clone [...];
cd audio-super-res;
```

### Contents

The repository is structured as follows.

* `./samples`: audio samples from the model
* `./src`: model source code
* `./docs`: html source code for the project webpage
* `./data`: code to download the model data

## Running the model

### Retrieving data

The `/data` subfolder contains code for preparing the VCTK speech dataset.

First, you need to download the data. Make sure you have enough disk space and bandwidth (the dataset is over 18G, uncompressed).

```
cd ./data/vctk;
make;
```

Next, you need to prepare data for training the model. 
Specifically, you need to create pairs of high and low resolution sound patches (typically, about 0.5s in length).
We have included a script called `prep_vctk.py` that does that.

There are two datasets that can be prepared. 
The single-speaker dataset consists only of VCTK speaker #1; it is relatively quick to train a model (a few hours). This option is great for quickly testing the model.
The multi-speaker dataset uses the last 8 VCTK speakers for evaluation, and the rest for training; it takes several days to train the model, and several hours to prepare the data.

The simplest way to prepare the data is to run `make` in the corresponding directory, e.g.
```
cd ./speaker1;
make;
```

Alternatively, you may directly use the script, which works as follows.

```
usage: prep_vctk.py [-h] [--file-list FILE_LIST] [--in-dir IN_DIR] [--out OUT]
                    [--scale SCALE] [--dimension DIMENSION] [--stride STRIDE]
                    [--interpolate] [--low-pass] [--batch-size BATCH_SIZE]
                    [--sr SR] [--sam SAM]

optional arguments:
  -h, --help            show this help message and exit
  --file-list FILE_LIST
                        list of input wav files to process
  --in-dir IN_DIR       folder where input files are located
  --out OUT             path to output h5 archive
  --scale SCALE         scaling factor
  --dimension DIMENSION
                        dimension of patches
  --stride STRIDE       stride when extracting patches
  --interpolate         interpolate low-res patches with cubic splines
  --low-pass            apply low-pass filter when generating low-res patches
  --batch-size BATCH_SIZE
                        we produce # of patches that is a multiple of batch
                        size
  --sr SR               audio sampling rate
  --sam SAM             subsampling factor for the data
```

The output of the data preparation step are two files in `.h5` format containing, respectively, the training and validation pairs of high/low resolution sound patches.

### Training the model

Running the model is handled by the `src/run.py` script.

```
usage: run.py train [-h] --train TRAIN --val VAL [-e EPOCHS]
                    [--batch-size BATCH_SIZE] [--logname LOGNAME]
                    [--layers LAYERS] [--alg ALG] [--lr LR]

optional arguments:
  -h, --help            show this help message and exit
  --train TRAIN         path to h5 archive of training patches
  --val VAL             path to h5 archive of validation set patches
  -e EPOCHS, --epochs EPOCHS
                        number of epochs to train
  --batch-size BATCH_SIZE
                        training batch size
  --logname LOGNAME     folder where logs will be stored
  --layers LAYERS       number of layers in each of the D and U halves of the
                        network
  --alg ALG             optimization algorithm
  --lr LR               learning rate
```

For example, to run the model on data prepared for the single speaker dataset, you may do the following.

```
python run.py train \
  --train ../data/vctk/speaker1/vctk-speaker1-train.4.16000.8192.4096.h5 \
  --val ../data/vctk/speaker1/vctk-speaker1-val.4.16000.8192.4096.h5 \
  -e 200 \
  --batch-size 64 \
  --lr 3e-4 \
  --logname singlespeaker
```

The above run will save its state in the folder `./singlespeaker.lr0.000300.1.g4.b64`.

### Testing the model

The `run.py` command may be also used to evaluate samples from the model.

```
usage: run.py eval [-h] --logname LOGNAME [--out-label OUT_LABEL]
                   [--wav-file-list WAV_FILE_LIST] [--r R] [--sr SR]

optional arguments:
  -h, --help            show this help message and exit
  --logname LOGNAME     path to training checkpoint
  --out-label OUT_LABEL
                        append label to output samples
  --wav-file-list WAV_FILE_LIST
                        list of audio files for evaluation
  --r R                 upscaling factor
  --sr SR               high-res sampling rate
```

In the above examples, we would run something along the lines of

```
python run.py eval \
  --logname ./singlespeaker.lr0.000300.1.g4.b64/model.ckpt-20101 \
  --out-label singlespeaker-out \
  --wav-file-list ../data/vctk/speaker1/speaker1-val-files.txt \
  --r 4
```

This will look at each file specified via the `--wav-file-list` argument (these must be high-resolution samples),
and create for each file `f.wav` three versions:

* `f.singlespeaker-out.hr.wav`: high resolution version (should be same as original)
* `f.singlespeaker-out.lr.wav`: low resolution version processed by the model
* `f.singlespeaker-out.sr.wav`: super-resoloved version

These will be found in the same folder as `f.wav`.

## Remarks

We would like to emphasize a few points.

* Machine learning algorithms are only as good as their training data. If you want to apply our method to your own old recordings, you will most likely need to collect your own labeled examples.
* You will need a very large model for large and diverse datasets (such as the 1M Songs Dataset, for example)
* Interestingly, super-resolution works better on aliased input (no low-pass filter). This is not reflected well in objective benchmarks, but is noticeable when listening to the samples. For applications like compression (where you control the low-res signal), this may be important.
* More generally, the model is very sensitive to how low-resolution samples are generated. Even the type of low-pass filter (Butterworth, Chebyshev) will affect performance.

### Extensions

The same architecture can be used on many time series tasks outside the audio domain. We have successfully used to impute functional genomics data, and denoise brain signal recordings. Stay tuned for more updates!

## Feedback

Send feedback to [Volodymyr Kuleshov](http://www.stanford.edu/~kuleshov).
