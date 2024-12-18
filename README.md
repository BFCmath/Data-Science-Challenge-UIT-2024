# Data-Science-Challenge-UIT-2024

## Table of Contents
- [Introduction](#introduction)
  - [Group B DSC UIT 2024](#group-b-dsc-uit-2024) 
  - [Evaluation Criteria](#evaluation-criteria)
  - [Terms and Conditions](#terms-and-conditions)
 - [Dataset](#dataset)
- [Public Test Results](#public-test-results)
- [Main Approaches](#main-approaches)
  - [ViLT Model](#vilt-model)
  - [Multilabel Classification Approach](#multilabel-classification-approach)
  - [Traditional Data Science Approach](#traditional-data-science-approach)
  - [Vintern Model](#vintern-model)
  - [One-vs-All Approach](#one-vs-all-approach)
  - [Stacking](#stacking)
- [Project Structure](#project-structure)
- [Private Test Results](#private-test-results)
- [Technologies Used](#technologies-used)
- [Acknowledgments](#acknowledgments)
## Introduction
This repository contains my submission for the **Group B** of the **DSC UIT 2024**, with the task of **Multimodal Sarcasm Detection on Vietnamese Social Media Texts**. In this challenge, teams are required to develop multimodal models capable of detecting sarcasm in Vietnamese social media posts that combine text and images.

### Group B DSC UIT 2024
**Challenge Topic:** Multimodal Sarcasm Detection on Vietnamese Social Media Texts

The goal is to propose a multimodal approach for sarcasm detection on a dataset of Vietnamese text-image posts gathered from social media platforms.

### Evaluation Criteria
Submissions are evaluated based on **Precision**, **Recall**, and **F1 score** for predicted labels against the provided labels. Rankings are based on the F1 score.

### Terms and Conditions
- Only the ViMMSD dataset is permitted for use.
- Public and private test data must not be manually annotated, nor should any data augmentation be applied.
- Only pre-trained models from the approved list are allowed.
- Teams must report the pre-trained embeddings and language models used.

### Dataset
- **Train, Dev, and Test Datasets**: Available on [Kaggle](https://www.kaggle.com) (search “DSC”) or by request.
- **Labels**: `multi-sarcasm`, `not-sarcasm`, `image-sarcasm`, and `text-sarcasm`.
- **Class Distribution** (imbalanced dataset):
  - **Train Set**: 10,805 instances (6062 `not-sarcasm`, 4224 `multi-sarcasm`, 77 `text-sarcasm`, 442 `image-sarcasm`)
  - **Test Set**: 1413 instances for public test, 1504 instances for private test

## Public Test Results
Summary of public test set results:

![public results](images/public_test_results.png)

For more details, refer to [this notebook](compare_val_test.ipynb) or [this CSV file](model.csv).

## Main Approaches 
### ViLT Model
- **ViLT Paper Summary**: [Summary Notebook](papers/ViLT.ipynb)
+ During testing, I observed that the ViLT inference code’s output was simply a softmax layer, where the node with the highest probability produced a one-word answer.
+ I modified this setup by replacing the linear layer with a softmax layer containing four output nodes, then fine-tuned the model on the training dataset using a CrossEntropy loss function.
+ Initially, the training dataset was split into training and validation sets in a stratified manner. However, I later used a [`GroupShuffleSplit`](manually_process/split_test_val.ipynb) approach to ensure that identical posts (those sharing the same caption) did not appear in both the training and validation sets.
+ The model’s input consists of the image caption (without any preprocessing) alongside the image itself.
+ Training began with the model’s layers frozen, allowing only the output layer to be trained. Once stable, I incrementally unfroze additional layers and reduced the learning rate for more refined fine-tuning.
+ Various combinations of learning rates, batch sizes, epochs, and layer-unfreezing strategies were tested, with the best F1 score achieved on the public test set reaching approximately 0.39.
+ For more details on these experiments, refer to [vilt v1](logs/vilt-v1.ipynb) and [vilt v2](logs/vilt-v2.ipynb).
+ Fine-tuning source code for ViLT is available on [Colab](https://drive.google.com/file/d/109Oerq2BfQdG6g3qEl0LBYiZMRbZMvsR/view?usp=sharing)

### Multilabel Classification Approach
- This approach was motivated by a few key observations:
    - The `multi-sarcasm` label includes both `text-sarcasm` and `image-sarcasm` components.
    - The `not-sarcasm` label includes both `not-text-sarcasm` and `not-image-sarcasm`.
    - The dataset is highly imbalanced.
    
- Given these factors, I decided on a multilabel classification approach:
    - The model now outputs two classes: `image-sarcasm` and `text-sarcasm`. Here, we assume that `multi-sarcasm` is the combination of both `text-sarcasm` and `image-sarcasm`, while `not-sarcasm` is represented by `not-text-sarcasm` and `not-image-sarcasm`.
    - This approach helps balance the previously imbalanced dataset.

- I adapted the ViLT model by modifying its output layer to a linear layer with two nodes for this multilabel setup.

- After reviewing [A Survey on Multilabel Learning in Deep Learning](papers/MultiLabelDL.ipynb), I tested various loss functions:
    - `Focal Loss` (for multilabel)
    - `BinaryCrossEntropy` (for multilabel)
    - `ZLPR` (for multilabel)
    - `WeightedCrossEntropy` (for multilabel - label powerset)
    - `F1 Soft` (for multilabel - label powerset)

- Using the same strategies applied to the base ViLT model, `Focal Loss` and `WeightedCrossEntropy (WCE)` produced the best results, with F1 scores of 0.38 and 0.39, respectively.

- For further details, you can check my notes on [ViLT with Focal Loss](logs/vilt-focal.ipynb) and [ViLT with Weighted CrossEntropy](logs/vilt-weighted-ce.ipynb).

### Traditional Data Science Approach
- Based on the assumption that icons and hashtags provide useful clues for prediction, I conducted an [analysis](manually_process/text_hidden_feature.ipynb) on the dataset to extract some [hidden features](emojis).
- I tested several traditional algorithms, including Logistic Regression, SVM, and tree-based models. However, the results were not promising, so this approach was discontinued.

### Vintern Model
- [Vintern Paper Summary](papers/VinTern-1B.ipynb)
- After testing various models, I selected Vintern as my Vision Language Model (VLM) based on several constraints:
    - Model size (1-2B) to fit resource limitations
    - Support for the Vietnamese language
    - Availability of public pretrained weights
    - A user-friendly API
- Unlike ViLT, Vintern offers a modern chat-response JSON format for inputs, which limited my ability to deeply modify the model (e.g., changing output layers). However, Vintern, along with the Hugging Face API, includes LoRA fine-tuning scripts, which significantly enhanced predictions when combined with instruction-based fine-tuning.

- Fine-tuning and prompt strategies improved the model's robustness:
    - [Prompt 1](logs/vintern-prompt1.ipynb)
    - [Prompt 2](logs/vintern-prompt2.ipynb)
    - [Prompt 3](logs/vintern-prompt3.ipynb)
    - [Prompt 5](logs/vintern-prompt5.ipynb)
    - [Prompt 6](logs/vintern-prompt6.ipynb)

- Most results achieved F1 scores between 0.4 and 0.41 for the best epoch across prompts.

- **Fine-tuning Strategy:**
    - Each prompt was trained one epoch at a time, saving the model after each epoch to manage GPU memory and reduce overfitting risks.
    - To optimize VRAM, only LoRA fine-tuning was used, and images were resized to smaller dimensions.
    - I utilized the same train-validation group [split data](data/train_val_group_split.txt) as for ViLT.
    - [JSONL extraction file for Vintern](extraction/jsonl_extraction.ipynb).

- **Prompt Explanations:**
    - **Prompt 1:** Simple prompt structure.
    - **Prompt 2:** Indicates that `multi-sarcasm` = `text-sarcasm` + `image-sarcasm`.
    - **Prompt 3:** Provides a detailed explanation of each label.
    - **Prompt 5:** Chain of Thought (CoT) prompt, guiding the model from text and image understanding to label inference.
    - **Prompt 6:** Similar to Prompt 2, but denotes icons and removes redundant dots, slashes ,commas from each word.

- For access to the Vintern fine-tuning script used in this competition, please contact me.
![Base Vintern](images/base-Vintern.png)

### One-vs-All Approach
- Building on the multilabel classification concept, I implemented a one-vs-all approach with Vintern to predict each label individually.
- Results for each label are available in the [result_ova folder](result_ova):
    - **Image OVA**: Predicts `image-sarcasm` vs. `not-image-sarcasm`
    - **Text OVA**: Predicts `text-sarcasm` vs. `not-text-sarcasm`
    - **Multi OVA**: Predicts `multi-sarcasm` vs. `not-multi-sarcasm`
    - **[Yes-No OVA](logs/vintern-yes-no.ipynb)**: Predicts `sarcasm` vs. `not-sarcasm`
    - **[Prompt 4](logs/vintern-prompt4.ipynb)**: Uses a Chain of Thought (CoT) prompt to predict `sarcasm` vs. `not-sarcasm`
    
- Among these, only the results from the **Yes-No OVA** and **Prompt 4** produced positive outcomes.

### Stacking

#### Stack OVA
- To enhance results, I explored a stacking approach by combining multiple models.
- First, I combined the results from various OVA models:
    - `text OVA` + `image OVA`
    - `text OVA` + `image OVA` + `multi OVA` + `yes-no OVA`
    - `multi OVA` + `yes-no OVA`
- I used Logistic Regression, SVM, and Random Forest to stack the OVA results and also manually wrote stacking rules. However, both approaches did not yield positive results.

#### Stack Base Vintern + OVA
- Next, I stacked the `yes-no OVA` results with the Base Vintern results by:
    - Counting the frequency of each pair of predicted labels from both models.
    - Manually evaluating these label pairs against the ground truth to assess correctness.
    - Identifying the pairs that yielded correct predictions and updating the Base Vintern results accordingly.
    - You can see the approach in [this notebook](merge_approach/merge_ova_train.ipynb).
![merge with yes no](images/merge_yes_no.png)
- As shown above, this approach improved the results!
- You can further explore this approach [here](merge_approach/ml_appoarch.ipynb).

#### Stack Base Vintern + Vintern with Image Sarcasm + OVA
- After additional testing, I overfitted the Base Vintern by training for more epochs (epochs 4, 5, 6, 7 for Vintern 1 and epochs 3, 4, 5 for Vintern 2).
- This decision was based on the assumption that the `yes-no OVA` could strengthen results for `not-sarcasm` and `multi-sarcasm`, so I focused on improving `image-sarcasm` and `text-sarcasm`.
- **Note**:
    - Text sarcasm instances were limited, making it challenging to improve results.
    - Attempts to increase epochs for text sarcasm alone negatively impacted the model.
    - I also experimented with Weighted CrossEntropy to emphasize `text-sarcasm`, but results did not improve.
- I then stacked the Base Vintern model with `yes-no OVA` and additional image-sarcasm models (e.g., `vintern_v1_epoch5`).
- Using the same strategies, I manually evaluated label pairs.

![merge with yes no and 2nd vintern](images/merge_best2_yes_no.png)

- This approach led to further improvements!
- You can explore this strategy in more detail [here](merge_approach/ml_appoarch.ipynb).

#### Stack Base Vintern + 2nd Vintern + Vintern with Image Sarcasm + OVA
- Continuing with the stacking concept, I attempted to merge three or more models with the `yes-no OVA`, including:
    - Models with high scores on the training set.
    - Models with high scores on the test set.
    - Models with stable scores on both train and test sets.
    - The top-performing models across train, test, and one model with consistent scores.
- Although many combinations did not perform well, some produced stable and high F1 scores.
- Below is the result for merging [Base Vintern] + [Vintern e1 epoch 5] + [Vintern e1.1 epoch 7] + [Yes-no epoch 2].
![merge 3 models with yes no](images/merge_b2_3_yes_no.png)

- You can check the results in [this notebook](merge_approach/merge_ova_train.ipynb).

#### Using ML Approach to Stack Models

- I also experimented with traditional machine learning methods to determine the optimal weights for each model in the stack.
- Details of this approach can be found [here](merge_approach/ml_appoarch.ipynb).
- However, due to instability in the results, I ultimately decided to discontinue this approach.

## Project Structure

- `data/`: Contains all data files and folders.
    - `csv/`: CSV files.
    - `json/`: JSON files.
    - `jsonl/`: JSONL files, primarily for Vintern.
    - `*-images/`: Image folders from the competition across 3 phases.
    - `resized_images/`: Resized images for Vintern fine-tuning.
    - `weights/`: Fine-tuned weights for ViLT and Vintern.
    - `train_val_group_split.txt`: File with train and validation IDs (split from the original training set of the competition).

- `emojis/`: All files related to icon-emojis.
- `extraction/`: Contains extraction-related files.
    - `jsonl_extraction/`: JSONL extraction for Vintern.
    - JSON file extraction for preprocessing labels for Vintern epoch 6.

- `images/`: Contains images used in the README.
- `logs/`: Logs for Vintern and ViLT fine-tuning.

- `manually_process/`: Contains files for manual processing.
    - `text_hidden_features`: Analysis information on labels.
    - `split_test_val.ipynb`: Splits train and validation IDs.

- `merge_approach/`: Files for evaluating and executing the stacking approach.
    - `merge_ova_test.ipynb` and `merge_ova_train.ipynb`: Evaluate stacking approach and save results to files.
    - `submission.ipynb`: Converts results to `results.json` and `results.zip` formats.
    - `merge_and_submit.ipynb`: End-to-end submission file; input is the file name, output is `results.zip`.
    - `log_merge_and_submit.txt`: Log file for `merge_and_submit.ipynb`.
    - `ml_approach.ipynb`: Uses machine learning for stacking models.

- `merge_test/` and `merge_val/`: Contains result files for merging in test and validation sets.
- `result_ova/` and `results_test_val/`: Result files for OVA and Base Vintern approaches.
- `submissions/`: Submission format files (`results.json` and `results.zip`).
- `result_private_test/` and `submissions_private_test/`: Vintern-related and merged results for the private test.

- `compare_val_test.ipynb`: Displays results for validation and public test sets.
- `private_test_analysis.ipynb`: Merges and saves results for the private test set.
- `models.csv`: CSV file with results on validation and public test sets.
- `web_image.html`: Hosts and renders images, mainly for image analysis.
## Private Test Results
Final model choices were based on stability and high F1 scores across train and test sets:

![best vintern results](images/final_best_vintern.png)

Selected Models:
- **vintern_v1_e3**: `te_b2_1epoch5_no_epoch2`
- **vintern_v1_1_e4**: `te_b2_1epoch5_b3_1.1_e7_no_e2`
- **vintern_v1_1_1_e6**: `te_b2_1epoch5_b3_1.1_e7_no_e2`

Final private test score: ~0.40 F1.

![private test results](images/private_test_results.png)

## Technologies Used
- [Kaggle](https://www.kaggle.com)
- [Google Colab](https://colab.research.google.com)
- PyTorch
- Hugging Face 

## Acknowledgments
- **Vintern**: [Paper](https://arxiv.org/pdf/2408.12480)
- **InternVL**: [Paper](https://arxiv.org/pdf/2312.14238)
- **Deep Learning for Multi-Label Learning: A Comprehensive Survey**: [Paper](https://arxiv.org/pdf/2401.16549)
- **ViLT**: [Paper](https://arxiv.org/pdf/2102.03334)