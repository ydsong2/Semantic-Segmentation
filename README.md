# Segmentation Wrap-up Report

# 1. Project Overview

We live in an era of mass production and mass consumption. While convenient, this lifestyle has led to various social issues, such as "waste crises" and "landfill shortages."

Proper waste sorting and recycling is one way to reduce this environmental burden. Well-sorted waste is recognized for its value as a resource and recycled, whereas incorrectly sorted waste is classified as general waste and ends up in landfills or incinerators.

Therefore, we aim to solve this problem by building a segmentation model that identifies waste in images. The provided dataset consists of images containing 11 categories of waste, including background, general waste, plastic, paper, and glass.

High-performance models created by you could be installed at waste disposal sites to assist with accurate sorting, or used for children's recycling education. Please save the Earth from crisis! 🌎

- **Input:** Images containing waste objects are used as model inputs. Segmentation annotations are provided in COCO format.
- **Output:** The model returns category values for each pixel coordinate. These are compiled into a CSV file matching the submission format.

---

# 2. Project Team Members and Roles

- **김동영:** Data and Inference Visualization, Pretrained Weight Search, Data Augmentation
- **김형훈:** Validation Set Search, Training Technique, Pseudo Labeling
- **송영동:** Using frameworks other than mmsegmentation
- **정상헌:** Model Search, Copy-Paste Augmentation
- **최휘준:** Model Search, Training Technique

---

# 3. Project Execution and Results

## I. Experiments

## 1) Validation Set Search

### ① Idea or Hypothesis
There was a significant gap between the baseline validation set's mIoU and the submission mIoU. Since we couldn't precisely determine the criteria used to create the baseline validation set, we decided not to use the default train/validation dataset split.

We hypothesized that the `train_all` and `test` datasets were sampled from the overall population, meaning they would not deviate significantly from the general distribution. We also observed that the mask sizes for each class were relatively similar.

Based on this hypothesis and observation, we believed that using `StratifiedGroupKFold` to split `train_all` into train and validation sets—while maintaining the class ratio—would yield a highly reliable validation set.

![Untitled](readme_img/Untitled.png)

### ② Experiment Design
Using sklearn's `StratifiedGroupKFold`, we divided the entire dataset into 5 folds. We decided to proceed with future experiments using the validation set (from the specific fold) that showed the smallest gap with the test mIoU. (Model used: fcn_r50)

### ③ Experiment Results

| Valid Fold Num. | Best Epoch | mAcc | mIoU (Valid) | Submission mIoU | Performance Gap |
| --- | --- | --- | --- | --- | --- |
| fold0 | 47 | 0.7285 | 0.6180 | 0.5430 | 0.0750 |
| fold1 | 44 | 0.6900 | 0.5830 | 0.5463 | 0.0367 |
| fold2 | 25 | 0.7216 | 0.5947 | 0.5401 | 0.0546 |
| fold3 | 27 | 0.6647 | 0.5621 | 0.5363 | 0.0258 |
| fold4 | 37 | 0.6896 | 0.5871 | 0.5573 | 0.0298 |

### ④ Results Analysis
The validation mIoU had a wide distribution ranging from 0.56 to 0.61, while the test mIoU ranged from 0.53 to 0.55. Based on the results, we determined that the Fold 3 validation set was the most similar to the test data and proceeded with the project using it.

However, while the validation mIoU and test mIoU were proportional up to a certain point, we noticed that beyond a certain threshold (0.74), the validation mIoU lost its correlation with the test score. 

The validation mIoU obtained using cross-validation would likely be the most accurate reflection of the test mIoU, but utilizing cross-validation was too time-consuming. Therefore, we decided to conduct all experiments using Fold 3. We also felt the need to study effective split methods specifically valid for segmentation rather than just relying on `StratifiedGroupKFold` (though resources on this were hard to find).

---

## 2) Pretrain Weight Search

### ① Idea or Hypothesis
Instead of merely loading the pretrained weights for the backbone, loading the full model weights (including the Decoder) pretrained on a specific dataset and then fine-tuning it should lead to faster and better convergence. Furthermore, the effectiveness will vary depending on the characteristics of the pretrained dataset.

### ② Experiment Design
- Models used: FCN-R101, HRNet-w48.
- All parameters other than the weights were kept identical for each model.
- We loaded different pretrained weights, trained the models, and recorded the trends.

### ③ Experiment Results

![W&B Chart 2023. 1. 9. 오전 11_37_26.png](readme_img/WB_Chart_2023._1._9._%25EC%2598%25A4%25EC%25A0%2584_11_37_26.png)

As hypothesized, there were significant differences in convergence speed and early training results depending on the pretrained weights used.

### ④ Results Analysis
1. Loading the entire model pretrained on a dataset with similar characteristics yields better results than just loading backbone weights.
2. The Pascal-Context dataset shares the most similar characteristics with this project's dataset; therefore, using weights pretrained on Pascal-Context is highly advantageous.
3. For datasets like Cityscapes, where the base image size ratios and overall characteristics differ greatly, it might actually be better *not* to use those pretrained weights.

---

## 3) Model Search

### ① Idea or Hypothesis
Through the UperNet_ConvNext experiment, we confirmed that higher model capabilities correlate with higher validation mIoU. This led to the hypothesis that models with a high "Paper mIoU" in the ADE20K semantic segmentation competition would also perform well on our dataset. Thus, we conducted a Model Search based on Paper mIoU.

### ② Experiment Design
1. Researched the Paper mIoU of all models in the MMSeg Config.
2. Trained and submitted MMSeg-provided models in descending order of their Paper mIoU.
3. Referenced *Papers with Code* to train and submit top-tier SOTA models.

![Untitled](readme_img/Untitled%201.png)

### ③ Experiment Results
[Segmentor list (ADE20K) (1)](https://www.notion.so/6fd9a8d3671a46c3b8ea2aeb5a3718a4)

### ④ Results Analysis
- Generally, Paper mIoU and test mIoU show a positive correlation.
- The Mask2Former model is our SOTA model.
- For the backbone, BEiT-Adapter showed the best performance.

---

## 4) Training Technique

### ① Idea or Hypothesis
The strong performance of `upernet_beit_adapter` (Rank 10 on Papers with Code for ADE20K) was not reproducing on our dataset. 

We noticed that UperNet's baseline learning rate (lr) defaults to 2e-5 with a warmup. We hypothesized that in the early stages—before gradients are properly oriented—a large learning rate might be destroying the pretrained backbone (BEiT).

### ② Experiment Design
As a control group, we trained the entire model with an lr of 6e-5. 
As an experimental group, we applied a training technique that differentiates the learning rates: 4e-6 for the backbone and 4e-5 for the decoder.

| Color | Description |
| --- | --- |
| Red (Control) | lr: 6e-5 |
| Blue (Experimental) | backbone lr: 4e-6, decoder lr: 4e-5 |

### ③ Experiment Results

![Untitled](readme_img/Untitled%202.png)
![Untitled](readme_img/Untitled%203.png)

### ④ Results Analysis
The results showed a massive difference in initial loss and mIoU for the model applying a smaller learning rate to the backbone. 

As an area for future improvement, we were unable to conduct a strict A/B test because we did not rigorously control the variables by setting the decoder's learning rate to exactly 6e-5 in the experimental group. 

We suspect the reason for this performance gap is that the pretrained BEiT model, having undergone self-supervised learning, already possesses the ability to identify object boundaries. Therefore, minimizing the updates to the backbone (as hypothesized) likely caused this difference. However, more systematic and precise experiments are needed to confirm the exact cause.

![Untitled](readme_img/Untitled%204.png)

---

## 5) Data Augmentation - Style

### ① Idea or Hypothesis
We used data augmentation, a proven method for boosting model performance in computer vision tasks.

### ② Experiment Design
1. Training without data augmentation.
2. Base augmentations (GaussNoise, RandomBrightnessContrast, HueSaturationValue).
3. Color augmentations (CLAHE, ColorJitter, HueSaturationValue).
4. RandomResizedCrop.

### ③ Experiment Results

![W&B Chart 2023. 1. 9. 오전 11_53_10.png](readme_img/WB_Chart_2023._1._9._%25EC%2598%25A4%25EC%25A0%2584_11_53_10.png)

None of the augmentation types yielded a meaningful performance improvement.

### ④ Results Analysis
Given the nature of this competition, the model heavily relies on color information to classify pixels into objects. Accordingly, we can deduce that data augmentations altering pixel RGB values had a negative impact. 

Therefore, we decided to explore new augmentation techniques rather than just altering the style of existing images.

---

## 6) Data Augmentation - Copy & Paste

### ① Idea or Hypothesis

![output.png](readme_img/output.png)

The provided data suffers from extreme class imbalance. Specifically, 'Clothing' and 'Battery' data are so scarce that the model fails to classify these classes properly.

### ② Experiment Design
1. From the entire dataset, select images with the highest proportion of background pixels to use as background images.
2. Copy only the masks of the objects from images containing Clothing and Battery.
3. Randomly scale and rotate these object masks, then paste them onto the background images.

![0011.png](readme_img/0011.png)

Examples of images created through this process are shown above. We added 50 augmented Battery images and 100 augmented Clothing images to the training set and proceeded with training.

### ③ Experiment Results

![W&B Chart 2023. 1. 9. 오후 12_04_25.png](readme_img/WB_Chart_2023._1._9._%25EC%2598%25A4%25ED%259B%2584_12_04_25.png)
![W&B Chart 2023. 1. 9. 오후 12_04_16.png](readme_img/WB_Chart_2023._1._9._%25EC%2598%25A4%25ED%259B%2584_12_04_16.png)

We confirmed that the model classified the Clothing and Battery classes much better when the data was augmented.

### ④ Results Analysis
We proved that this augmentation method is effective in resolving class imbalance. 

However, compared to Clothing, the Battery class showed faster convergence but no significant increase in maximum performance. The reason can be seen in the example below:

![Untitled](readme_img/Untitled%205.png)

The train dataset mostly contains cylindrical batteries, while cuboid batteries (like the one above) only existed in the test dataset. Thus, even with augmentation, it was practically impossible for the model to correctly classify such images. Moreover, because the absolute number of Battery data points was so small, edge cases like this heavily restricted the IoU metric from rising above a certain level.

---

## 7) Pseudo Labeling

### ① Idea or Hypothesis
We assumed that using pseudo-labels generated by our best mIoU model would lead to further performance improvements. 

We hypothesized that if the score increased, we could iteratively generate pseudo-labels with the new best model and train sequentially, eventually allowing the model to train on highly accurate pseudo-labels.

### ② Experiment Design
Model: `upernet_beit_adapter_large`

| Model | Description |
| --- | --- |
| 1 | (Control) Model trained on the train set that achieved an mIoU of 0.7773. |
| 2 | Model trained using pseudo-labels generated by Model 1. |
| 3 | Model trained using pseudo-labels generated by Model 2. |

### ③ Experiment Results

| Model | mIoU |
| --- | --- |
| 1 | 0.7773 |
| 2 | 0.7959 |
| 3 | 0.7929 |

### ④ Results Analysis
We observed a significant performance boost (+0.02) when using pseudo-labels. 
However, contrary to our expectations, retraining with pseudo-labels generated by the even more accurate model (Model 2) did not show a significant difference in performance (0.7959 vs 0.7929). 

We suspect this might be because we only trained it for 5 epochs (a very small number). (We could not conduct further experiments due to lack of time). 

Overall, it was an experiment where we directly confirmed the effectiveness of pseudo-labeling.

---

## 8) Ensemble

### ① Idea or Hypothesis
Combining various inference results generated by models with different characteristics (architectures, pretrained weights, training parameters, datasets used) through ensembling will yield highly generalized submission results.

### ② Experiment Design

![Untitled](readme_img/Untitled%206.png)

We gathered various high-performing inference files with diverse characteristics, combined them using a hard-vote ensemble with different configurations, and recorded the submission results.

### ③ Experiment Results

![Untitled](readme_img/Untitled%207.png)

### ④ Results Analysis
Generally, ensembling consistently resulted in an increased submission score. 

However, it was difficult to establish qualitative metrics, such as identifying exactly which files guaranteed a score increase when ensembled. This seems to stem from the inherent characteristics (excessive randomness) of the ensemble technique itself.

---

## II. Utilities

The following are utilities developed by the team during the project to ensure efficient execution of tasks.

| Index | Filename | Utility Description |
| --- | --- | --- |
| 1 | make_pseudo_label.py | Utility that creates pseudo-labels (image annotations) for test data using a trained model, formatted for use in mmsegmentation. |
| 2 | split_train_valid.ipynb | Utility that splits `train_all` into train/valid sets while maintaining the class distribution. |
| 3 | convert_mmseg_dataset.py | Utility that converts a COCO dataset into a dataset format compatible with mmseg. |
| 4 | copy_and_paste_augmentation.ipynb | Data augmentation utility that generates new images by masking images of certain classes and pasting them onto background images. |
| 5 | hard_vote_ensemble.ipynb | Utility to perform pixel-level hard-vote ensembling by taking multiple inference results. |
| 6 | streamlit visualization | Streamlit utility to visually verify the model's inference results on the test dataset. |
