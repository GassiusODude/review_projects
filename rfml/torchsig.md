# Torch Sig

[Torch Sig](https://github.com/TorchDSP/torchsig) by TorchDSP

TorchSig is an open source RFML library on GitHub.  It supports the simulation of radio-frequency (RF) signals to train machine learning (ML).

According to this [GRCon 2025 PDF](https://events.gnuradio.org/event/26/contributions/752/attachments/220/586/TorchSig_GRCon2025.pdf)

| Model | Description |
| --- | --- |
| detect.pt | YOLOv8x model for detection in Spectrogram |
| xcit.ckpt | An XCiT transformer model for narrowband modulation recognition |
| 11s.pt | YOLOv11 size 'S' model for energy detection in a wideband cut. |

## XCiT Classifier

In [L. Boegner, M. Gulati, G. Vanhoy, P. Vallance, B. Comar, S. Kokalj-Filipovic, C. Lennon, and R. D. Miller, "Large Scale Radio Frequency Signal Classification", Jul 2022](https://arxiv.org/abs/2207.09918), the XCiT transformer architecture is applied to modulation classification with Sig53 data set.  The XCiT architecture is a Tranformer based architecture used in image analysis.

* Signal classification of Sig53 dataset
* Compares `EfficientNet` and `XCiT` performance
* XCiT achieved highest accuracy at 71.16 % and EfficientNet at 69.73 %
* Confusion is usually within the same family.  With modulation family classification, the accuracy is close to 97 - 100 %.

The `xcit.ckpt` from release v1.1.0 is from May 2025.  That's 3 years after the paper.  Instead of the 53 signal classes, there seems to be 57 classes (added tones and chirps).  I also attempted to grab from the bucket as described in [Issue 271](https://github.com/TorchDSP/torchsig/issues/271)

### Torch-Sig GUI

I started a [TorchSig-GUI](https://github.com/GassiusODude/torchsig-gui) project to support a Web-based UI to configure a dataset for training.  This is currently using FastAPI to host a webpage and save a YAML configuration.  A script is provided to support creating the training and validation dataset.

Once the configuration is saved, I wanted to make use of this configuration.  I extracted the loss and classifier architecture from [classifier.ipynbc](https://github.com/TorchDSP/torchsig/blob/main/examples/classifier_example.ipynbc).  My initial thought was to use the `torchsig.utils.yaml` module to load the configured YAML.  However, the function would use a fixed call to `TorchSigIterableDataset`, leading to loading all signals (57) signals as potential generators.  This deviates from the goal of using the GUI to select the exact signals to support.

I generate two files to support this:

* classifier_module.py
  * This holds the Classifier architecture and loss functions
* script_gen_dataset.py
  * This holds a function to generate the dataset based on the YAML configuration.
  * It supports training my classifier and testing on the validation dataset to evaluate the trained classifier.

I was getting really poor performance with the full signal set (57 signal types).  I simplified to family classification and removed the OFDM, Linear FM, and tone.  I'm observing my AM modulation family classification has low precision and recall.  In the confusion matrix, it is labeled as other classes (like ASK/PSK/QAM).  At first, I thought it was using single-sideband AM on a digital modulations.  A classifier should confuse the two signals in that case.  Upon closer inspection, the AM generator is implemented as a bandwidth-filtered white noise system.

* Enforced limits - These are some limitations I'm putting into my configuration to simplify the classifier
  * `num_signals_min` = `num_signals_max` = 1 and `cochannel_overlap_probability` = 0.0
    * Just one label, no overlapping mixed of signals
  * `signal_center_freq_min` and `signal_center_freq_max` are both low
    * I'm not trying to do signal detection and classification at the same time.
* Limitations in RF signal simulation - These will impact the performance of this classifier on real-world examples.
  * Amplitude Modulation
    * With speech as input, the silence in speech would show up as tone/silence depending if the AM carrier is suppressed or not.  The implementation using bandwidth filtered white-noise will not match the AM speech signals.
  * Frequency Shift Keying
    * The simulator enforces an integer oversampling rate, meaning a rectangular pulse would have exactly the right number of samples in each symbol period.
    * I see MSK/GMSK uses modulation index of 0.5 but general FSK is either modulation index of 1.0 or ranges from 0.7 to 1.01.  The modulation indices are hard-coded and cannot be accessed outside of the generator.  This may limit the classifier from obvious FSK (with higher modulation index).
