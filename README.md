# Deep-Learning Super-resolution Image Reconstruction (DSIR)

######  | :file_folder: [Files](#files) | :computer: [Code](#code)| :memo: [Results](#results) |:speech_balloon: [Discussions](#discussions) | :copyright: [References](#references) |						

___

Super-resolution microscopy techniques (PALM, STORM…) can improve spatial resolution far beyond diffraction limit trough recording many frames of sparse single events emission (fluorescence labels). Low density (LD) data acquisition can provide tens of thousand of images of sparse events that can be readily localized using standard fitting algorithms (e.g, Thunder-STORM). However, the acquisition of such large number of images takes some time and can result to the sample drift and degrade the sample due to the intense light exposion. High density (HD) data of several hundred images can be faster in acquisition time but result of a very dense number of events per frames, which compromise the performance of the fitting localization algorithms. This repository proposes a method that use convolution neural network (ConvNet) auto-encoder [[1](#References)] to reconstruct a localization images from HD datasets. 



![convnet_autoencoder](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/autoencoder.png)

**Fig.1** - ConvNet auto-encoder representation for super-resolution image reconstruction.



>  These work were probably developed in parallel with the work of Elias Nehme, et all. published in [[2](#References)]. Both achieved very similar results despite different coding languages. 



ConvNet auto-encoder proposed here were coded in [PyTorch](http://pytorch.org).  We define an auto-encoder architecture which consider and input array of size (208, 208, 1), original image size of **26x26 px** resized by a factor of 8. The ground-truth label is a pixel converted image (208, 208, 1) from the list emitters positions (x, y). Both transformations are defined in  `data_load.py` file. 



![](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/visdom.png)

**Fig.2** - (Upper row) Training dataset ground-truth pixel localization image examples and (Lower row) correspondent trained auto-encoder predictions.



## Files 

- **`data_load.py`** : Parse the raw data: Rescale input image, plot position to pixel map label and convert both (data and label) to torch tensor.
- **`conv_autoencoder.py`**: training the ConvNet auto-encoder.
- **`load_model.py`**: load a pre-trained model and test against the example [Tubes HD dataset](http://bigwww.epfl.ch/smlm/challenge2013/datasets/Bundled_Tubes_High_Density/index.html). 



## Code

### Training Dataset

A dataset of randomly generated single emission events were used to training our model. [ThunderSTORM](https://github.com/zitmen/thunderstorm) [imageJ](https://imagej.net/Welcome) plugin data generator were used to generate training and validation datasets with the following settings:

| General setup:                         | Camera setup:                      |
| -------------------------------------- | ---------------------------------- |
| size: 26x26 px                         | Pixel Size: 100 nm                 |
| train set: **3500**, val set: **1500** | Photoelectrons per A/D counts: 1.0 |
| PSF: Integrated Gaussian               | Base level: 100                    |
| FWHM range: 200:300 nm                 |                                    |
| Intensity range: 80:2050 photons       |                                    |
| Density: 2.0 emitters/$`\mu m^2`$        |                                    |
| BG noise: 20                           |                                    |

You can also download the used dataset [here](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/dataset.zip).  The raw dataset should be uncompressed  at `~/data/dataset/` folder.  



### Auto-encoder

The 3 layers Encoder and 4 layers Decoder are defined following the code bellow:

```python
def forward(self, x):
    # Encode
        x = self.conv1(x)
        x = self.pool(F.relu(self.bn1(x)))  # out [16, 104, 104, 1]
        x = self.conv2(x)
        x = self.pool(F.relu(self.bn2(x))) # out [8, 52, 52, 1]
        x = self.conv3(x)
        x = self.pool(F.relu(self.bn2(x))) # out [8, 26, 26, 1]
	# Decode
        x = self.convt1(x)
        x = F.relu(self.bn2(x)) # out [8, 52, 52, 1]
        x = self.convt2(x)
        x = F.relu(self.bn1(x)) # out [16, 104, 104, 1]
        x = self.convt3(x)
        x = F.relu(self.bn2(x)) # out [8, 208, 208, 1]
        x = self.conv4(x)
        x = F.relu(self.bn4(x)) # out [1, 208, 208, 1]
```



### Loss function

Using an Adam optimizer, the loss function were defined as:

$`
loss(x, \hat{x}) = \frac{1}{N} \displaystyle \sum_{i=1}^N  |\hat{x_i} \otimes g - x_i \otimes g |^2 + 1e^{-5}*|\hat{x}i|^2
`$

where, $`x`$ is the label images, $`\hat{x}`$ is the neural network predictions, $`g`$ is a Gaussian kernel and $`N`$ is the total number of images per batches. The operation $` x \otimes g`$ denotes a 2D Gaussian convolution between $`x`$ and $`g`$. 



### Training Parameters

The _train()_ function in `conv-autoencoder.py` takes the following arguments:

- **epochs** (int): total number of training epochs. [100]
- **lr** (float): learning rate for the Adam optimizer. [1e-4]
- **batch_size** (int): batch size of training and validation dataset. [32]
- **seed**: randomization seed number. [99]
- **kernel_width** (int): size in pixel of the square Gaussian kernel ($`g`$). [5]
- **kernel_fwhm** (int): Full width of half maximum of the Gaussian kernel ($`g`$).  [3]
- **verbose** (boolean): defines whether show up the visdom output results. [True]
- **save** (boolean):  defines whether save or not the training model on the end (or KeyboardInterrupt) on training. [True]
- **model_path** (path): path where to save the model. [None]

The preset parameters will save the model on path:

`~/data/temp/_timestamp_/cae_model__timestamp_.pt` 

where _ṭimestamp_ = day month year _ hour minute second  string.



### Visdom 

We use [visdom](https://github.com/facebookresearch/visdom) to track the results during the training process. You need to run visdom server beforehand:

```python
python -m visdom.server
```

then you can open in your browser http://localhost:8097/# .

For each training epoch you you'll have a set of 4 images with the prediction image reconstruction and the correspondent pixel ground-truth localization and the loss function value plot.

![visdom_output](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/visdom_window.png)

**Fig.3** - Ground-Truth pixel localization, auto-encoder prediction image reconstruction and training loss visdom output.



## Results

We trained a model for 100 epochs using the [train dataset](#Training Dataset) and later a fine tuning with a smaller learning rate and a kernel_fwhm = 1. Resulted model can be found on `model/autoencoder_model.pt`. 



### Test Dataset

In order to test our trained model we used the  [Tubes HD dataset](http://bigwww.epfl.ch/smlm/challenge2013/datasets/Bundled_Tubes_High_Density/index.html) available from the 2013 IEEE International Symposium on Biomedical Imaging Challenge [[3](#References)]. ([Download Here](http://bigwww.epfl.ch/smlm/challenge2013/datasets/Bundled_Tubes_High_Density/sequence.zip))



### localization Image Reconstruction 

![single frame](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/single_frame.png)

**Fig.4** - Tubes HD dataset. (**Left**) single input frame image (**Right**) Zoom at red square and the ground-truth emitters positions(red crosses).

![single frame localization](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/single_frame_localization.png)

**Fig.5** - (**Left**) Single frame DSIR and (**Right**) comparison with ground-truth emitters positions (red crosses).

![single frame](https://github.com/leaxp/Deep-Learning-Super-Resolution-Image-Reconstruction-DSIR/tree/assets/localization.png)

**Fig.6** -  (**Left**) Reconstruction of the 361 frame of the Tubes HD dataset. (**Right**) Zoom (green square Fig.4) image and ground-truth emitters position (red dots). The total reconstruction time for all frames is about **3 sec**. 



## Discussions

Here we presented a ConvNet auto-encoder model applied to high density recorded stochastic super-resolution microscopy data. Although not discussed in here, this DSIR can improve performance compared with the state-of-the-art fitting algorithms that usually performed poorly with high density data. Also DSIR can be much faster the the most of fitting algorithms. 

The current performance might be improved by feeding the same auto-encoder with different training dataset, E.g., experimental measured sparse data where the label positions are define via a standard fitting algorithm. Despite, DSIR be able to reconstruct localization super-resolution images, it lacks in terms of quantitative information since the ConvNet auto-encoder doesn't outputs the localization coordinates of each detected event. 



## References

1. https://blog.keras.io/building-autoencoders-in-keras.html
2. https://arxiv.org/abs/1801.09631v2
3. http://bigwww.epfl.ch/smlm/challenge2013/index.html




![forthebadge](https://forthebadge.com/images/badges/made-with-python.svg) ![forthebadge](https://forthebadge.com/images/badges/built-with-science.svg)