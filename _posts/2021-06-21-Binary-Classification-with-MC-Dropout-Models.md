---
published: true
layout: post
title: Binary Classification with MC-Dropout Models
img: cat_vs_dogs.jpg
tags:
  - Uncertainty
  - Bayesian Deep Learning
  - Deep Learning
  - Machine Learning
  - Artificial Inteligence
---
## Binary Classification with MC-Dropout Models

An introduction to classification with Bayesian Deep Learning with the Monte-Carlo Dropout implementation. 

**Check the [interactive version](https://wandb.ai/sborquez/her2bdl/reports/Binary-Classification-with-MC-Dropout-Models--VmlldzozMTg0OTE) on the Weight & Biases plataform**.

### Introduction

Bayesian Deep Learning (BDL)[1] allows to include the uncertainty measurement for Deep Learning (DL) models. Quantify the confidence of the model may play an important role in the context of risk management. Different BDL methods have been developed in recent years to estimate uncertainty [2,3]. Still, to understand the mechanism and interpretation of the aleatoric and epistemic uncertainty [4], we first analyze them in a simpler and risk-less context, the **cats vs. dogs image classification.**


![](https://api.wandb.ai/files/sborquez/images/projects/131709/a91e89f8.png)

The proposed submodels [implementation](https://github.com/sborquez/her2bdl/tree/dev) of the Monte-Carlo dropout [2], allows us to use any pre-trained model as the encoder model _E_, for this dataset, it is convenient to use a well-known model for image classification. In particular, we use the Keras implementation of the **EfficientNet**, with the configuration **B0** and the weights trained with the **ImageNet dataset** (https://image-net.org/).
For the stochastic classifier model _C_, we implement it by stacking three fully dense connection layers. As the MC-Dropout model requires, a dropout layer is added between each dense layer to enforce the stochastic output.  The following figure shows the architecture of the model used for this binary classification problem.

![nn-EfficientNet.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/nn-EfficientNet.png)

In the rest of this post, we will discuss the training procedure, the classification performance and analyze and interpretation of the uncertainty.

### Training Procedure

The most important feature of an MC-Dropout model is the seamless integration with the most commonly used deep learning frameworks, especially the no alteration of the training procedure. It can fit its weight by the classics backpropagation algorithms. The only modification in the architecture is the addition of the dropout layer, which is trained as usual, i.e., for each step, we take a sample of the weights by dropping some connections on the network and propagating the error through the remaining connections.
The model is trained with the RMSProp algorithm, with a learning rate of 0.001, preliminary experiments have shown that the SGD and moment base backpropagate algorithms results in underfitting and models with a low generalization.

### Classification Metrics

In general, the performance metrics show that this model can perfectly classify images of cats and dogs.

![binary_training.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/binary_training.png)
![clasify.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/clasify.png)

### Prediction Examples

An MC-Dropout model performs the prediction of the class for a new input differently from a classic model of DL. The predictive distribution, i.e., the estimated distribution of classes for a given input, is calculated by sampling the trained model with _T_ stochastic forward passes and average the results. 

![predictive.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/predictive.png)

We implement this estimation with the proposed **2-step predictive distribution** algorithm with GPU accelerated Monte-Carlo forward passes. The following figures consist of the input image with the true and predicted classes (top left), the predictive distribution and epistemic uncertainty (top right), and a stochastic forward passes distribution by class.


![High mutual information_0_18.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/High mutual information_0_18.png)


### Epistemic Uncertainty

The previous section has shown that the model can classify the images, but we are interested in measuring uncertainty. We start with **epistemic uncertainty**.  This kind o uncertainty represents the ignorance of the model parameters; it's also referred to as the **model uncertainty** because it can be interpreted as the **confidence in the prediction**. Epistemic uncertainty can reduce this type of uncertainty by adding new samples to our datasets.

We measure the epistemic uncertainty with the **Predictive Entropy** (H) or the **Mutual Inforation** (I):

![epistemic.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/epistemic.png)

Predictive entropy _H_ capture the uncertainty in the prediction, whereas the mutual information _I_ captures the model’s confidence in its output.

We evaluate the best model obtained in the training phase on the test set.  The next figure shows samples from the evaluation set with higher predictive entropy and mutual information.

![high_entropy.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/high_entropy.png)
![high_mutual.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/high_mutual.png)

The epistemic uncertainty results show that  the mean prdictive entropy and mutual information for the evaluation set are near 0, so broadly speaking, the model has a high level of confidence, without any class preference.  Nonetheless, some outliers have a large epistemic uncertainty; for those samples (see the previous figure), the model has a low confidence level in its prediction.

### Aleatoric Uncertainty

The other type of uncertainty is the **aleatoric uncertainty**, also known as **data uncertainty**. This is an irreducible type of uncertainty and is **the noise inherent in the observations**. Aleatoric uncertainty is categorized into homoscedastic uncertainty, the uncertainty that stays constant for different inputs, and heteroscedastic uncertainty that depends on the inputs to the model, and can vary with each new image, thus some input will have higher uncertainty than others. Heteroscedastic uncertainty is especially important for computer vision applications.
By applying the method discussed in the paper [3], we can measure  for each input sample the aleatoric uncertainty, the **predictive variance**, building the **Aleatoric Model**. By moddifying the stochastic classifier from the MC-Dropout model, removing the MC-Dropout layers and adding the predictive variance ($\hat{\sigma}$) to the model's output. This method works by adding noise  from a normal distribution ϵ∼N(0,I) to the model output in previous to the softmax function, i.e. we corrupt the output in the logits space.

![ale.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/ale.png)

The next figure shows the new aleatoric model architecture, we can reuse the same weights from the encoder.

![nn-Aleatoric EfficientNet.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/nn-Aleatoric EfficientNet.png)

In order to training, the model uses the following loss function instead of the cross-entropy loss.

![loss.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/loss.png)

Notices that for this model _w_ is fixed, so the model's outputs are deterministic. Instead, we obtain stochastics output in the loss function evaluation.

We train the aleatoric model and was evaluate in the test set.

![binary_training_aleatoric.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/binary_training_aleatoric.png)


A higher value of predictive variance means a higher noise in a given input image. The next figure shows samples from the evaluation set with the lower and higher aleatoric uncertainty.  

![Low predictive_variance_0_60.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/Low predictive_variance_0_60.png)

![High predictive_variance_0_6.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/High predictive_variance_0_6.png)


The aleatoric uncertainty results show a difference in the distribution of uncertainty for each class. The dog class tends to have a higher uncertainty than the cat class.

![variance.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/variance.png)


### The Importance of Uncertainty

he next figure shows some relevant examples from the evaluation set with the higher **predictive entropy**.

![collage_high_predictive_entropy.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/collage_high_predictive_entropy.png)

The model has low confidence with these examples, and we can infer that three types of conditions may cause this. The first row shows that an image with a small pixel region containing factual information is a source of uncertainty. As humans, we also have some difficulty distinguishing these types of pictures due to the image's low resolution.

The second row contains images out of the distribution of the training set. Pictures of people or text are a real problem, primarily when we can found this type of error in official datasets used by the community.

Finally, the third row contains images of cats and dogs in a weird-looking pose, like the first picture, with a dog looking at the camera that actually is the top of a cat's head. Or cat and dog with mixed features, like a pointy ear dog and a pitbull fur cat.

![collage_predictive_variace.png](https://raw.githubusercontent.com/sborquez/sborquez.github.io/master/_drafts/collage_predictive_variace.png)

In the case of the predictive variance, samples with a large black region also present an high uncertainty. The difference between clases can be explained by the variability of the features of dogs in contrast to the low variability of the cat features. The differences between dogs' breeds, shapes, sizes, and furs are visually different from those of cats. Thus, creating more noise datasets for the dog class.

We can use these insights and apply them in other high-risk scenarios as image classification for medical diagnosis or stock price regression with uncertainty measurement. In the same line, we have been working in the [her2bdl](https://github.com/sborquez/her2bdl/tree/dev) repository. We aim to applying the BDL framework to the HER2 Scoring problem, evaluating expression of the human epidermal growth factor receptor 2 (HER2) by automatic examination of digital microscope images with immunohistochemistry (IHC) test on invasive breast cancer (BCa). 

If you are interested in the Her2 Scoring problem with BDL, check the interactive [reports](https://wandb.ai/sborquez/her2bdl/reports/) and paper (work in progress).

### References

* [1] Gal, Y. (2016). Uncertainty in deep learning. University of Cambridge, 1(3), 4.
* [2] Gal, Y., & Ghahramani, Z. (2016, June). Dropout as a bayesian approximation: Representing model uncertainty in deep learning. In international conference on machine learning (pp. 1050-1059). PMLR.
* [3] KENDALL, Alex; GAL, Yarin. What uncertainties do we need in bayesian deep learning for computer vision?. arXiv preprint arXiv:1703.04977, 2017.
* [4] DER KIUREGHIAN, Armen; DITLEVSEN, Ove. Aleatory or epistemic? Does it matter?. Structural safety, 2009, vol. 31, no 2, p. 105-112.


