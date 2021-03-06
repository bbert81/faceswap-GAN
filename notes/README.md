# Notes:
## In this page are notes for my ongoing experiments and failed attmepts.
### 1. BatchNorm/InstanceNorm: 
Caused input/output skin color inconsistency when the 2 training dataset had different skin color dsitribution (light condition, shadow, etc.). But I wonder if this will be solved after further training the model.

### 2. Perceptual loss
Increasing perceptual loss weighting factor (to 1) unstablized training. But the weihgting [.01, .1, .1] I used is not optimal either.

### 3. Bottleneck layers
~~In the encoder architecture, flattening Conv2D and shrinking it to Dense(1024) is crutial for model to learn semantic features, or face representation. If we used Conv layers only (which means larger dimension), will it learn features like visaul descriptors? ([source paper](https://arxiv.org/abs/1706.02932v2), last paragraph of sec 3.1)~~ Similar results can be achieved by replacing the Dense layer with Conv2D strides 2 layers (shrinking feature map to 1x1).

### 4. Transforming Emi Takei to Hinko Sano
Transform Emi Takei to Hinko Sano gave suboptimal results, due to imbalanced training data that over 65% of images of Hinako Sano came from the same video series.

### 5. About mixup and LSGAN
**Mixup** technique ([arXiv](https://arxiv.org/abs/1710.09412)) and **least squares loss** function are adopted ([arXiv](https://arxiv.org/abs/1712.06391)) for training GAN. However, I did not do any ablation experiment on them. Don't know how much impact they had on the outputs.

### 6. Adding landmarks as input feature
Adding face landmarks as the fourth input channel during training (w/ dropout_chance=0.3) force the model to learn(overfit) these face features. However it didn't give me decernible improvement. The following gif is the result clip, it should be mentoined that the landmarks information was not provided during video making, but the model was still able to prodcue accurate landmarks because similar [face, landmarks] pairs are already shown to the model during training.
  - ![landamrks_gif](https://www.dropbox.com/s/ek8y5fued7irq1j/sh_test_clipped4_lms_comb.gif?raw=1)

### 7. **Recursive loop:** Feed model's output image as its input, **repeat N times**.
  - Idea: Since our model is able to transform source face into target face, if we feed generated fake target face as its input, will the model refine the fake face to be more like a real target face?
  - **Version 1 result (w/o alpha mask)** (left to right: source, N=0, N=2, N=10, N=50)
    - ![v1_recur](https://www.dropbox.com/s/hha2w2n4dh49a1k/v1_comb.gif?raw=1)
    - The model seems to refine the fake face (to be more similar with target face), but its shape and color go awry. Furthermore, in certain frames of N=50, **there are blue colors that only appear in target face training data but not source face.** Does this mean that the model is trying to pull out trainnig images it had memoried, or does the mdoel trying to transform the input image into a particular trainnig data?
  - **Version 2 result (w/ alpha mask)** (left to right: source, N=0, N=50, N=150, N=500)
    - ![v2_recur](https://www.dropbox.com/s/zfl8zjlfv2srysx/v2_comb.gif?raw=1)
    - V2 model is more robust. Almost generates the same result before/after applying recursive loop except some artifacts on the bangs.

### 8. **Code manipulation and interpolation**: 
  - ![knn_codes](https://www.dropbox.com/s/a3o1cvqts83h4fl/knn_code_fit.jpg?raw=1)
  - Idea: Refine output face by adding infromation from training images that look like the input image.
  - KNN takes features extracted from ResNet50 model as its input.
  - Similar results can be achieved by simply weighted averaging input image with images retrieved by kNNs (instead of the code).
  - TODO: Implement **alphaGAN**, which integrates VAE that has a more representative latent space.

### 9. **CycleGAN experiment**:
  - ![cyckeGAN exp result](https://www.dropbox.com/s/rj7gi5yft6yw7ng/cycleGAN_exp.JPG?raw=1)
  - Top row: input images.; Bottom row: output images.
  - CycleGAN produces artifacts on output faces. Also, featuers are not consitent before/after transformation, e.g., bangs and skin tone.
  - ~~**CycleGAN with masking**: To be updated.~~

### 10. **(Towards) One Model to Swap Them All**
  - Objective: Train a model that is capable of swapping any given face to Emma Watson.
  - `faceA` folder contains ~2k images of Emma Watson.
  - `faceB` folder contains ~200k images from celebA dataset.
  - Hacks: Add **domain adversaria loss** on embedidngs (from [XGAN](https://arxiv.org/abs/1711.05139) and [this ICCV GAN tutorial](https://youtu.be/uUUvieVxCMs?t=18m59s)). It encourages encoder to generate embbeding from two diffeernt domains to lie in the same subspace (assuming celebA dataset covers almost the true face image dsitribution). Also, heavy data augmentation (random channel shifting, random downsampling, etc.) is applied on face A to pervent overfitting.
  - Result: Model performed poorly on hard sample, e.g., man with beard.

### 11. **Face parts swapping as data augmentation**
  - ![](https://www.dropbox.com/s/1l9n1ple6ymxy8b/data_augm_flowchart.jpg?raw=1)
  - Swap only part of source face (mouth/nose/eyes) to target face, treating the swapped face as a augmented training data for source face.
  - For each source face image, a look-alike target face is retrieved by using knn (taking a averaegd feature map as input) for face part swapping.
  - Result: Unfortunately, the model also learns to generates artifacts as appear in augmented data, e.g., sharp edges around eyes/nose and weirdly warped face. The artifacts of augmented data are caused by non-perfect blending (due to false landmarks and bad perspective warping).

### 12. Neural style transfer as output refinement
  - Problem: The output resolution 64x64 is blurry and sometimes the skin tone does not match the target face. 
  - Question: Is there any other way to refine the 64x64 output face so that it looks natural in, say, a 256x256 input image except increasing output resolution (which leads to much longer training time) or training a super resolution model?
  - Attempts: **Applied neural style transfer techniques as output refinement**. Hoping it can improve output quality and solve color mismatch without additional training of superRes model or increasing model resolution. 
  - Method: We used implementation of neural style transfer from [titu1994/Neural-Style-Transfer](https://github.com/titu1994/Neural-Style-Transfer), [eridgd/WCT-TF](https://github.com/eridgd/WCT-TF), and [jonrei/tf-AdaIN](https://github.com/jonrei/tf-AdaIN). All repos provide pre-trained models. We fed swapped face (i.e., the output image of GAN model) as content image and input face as style image.
  - Results: Style transfer of Gatys et al. gave decent results but require long execution time (~1.5 min per 256x256 image on K80), thus not appplicable for video conversion. The "Universal Style Transfer via Feature Transforms" (WCT) and "Arbitrary Style Transfer in Real-time with Adaptive Instance Normalization" (AdaIN) somehow failed to preserve the content information (perhaps I did not tune the params well).
  - Conclusion: **Using neural style transfer to improve output quality seems promising**, but we are not sure if it will benefit video quality w/o introducing jitter. Also the execution time is a problem, we should experiment with more arbitrary style transfer networks to see if there is any model that can do a good job on face refinement within one (or several) forward pass(es).
  - ![style_transfer_exp](https://www.dropbox.com/s/r00q5zxojxjofde/style_transfer_comp.png?raw=1)
