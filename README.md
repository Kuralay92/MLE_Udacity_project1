# Optimizing an ML Pipeline in Azure

## Table of contents
   * [Overview](#Overview)
   * [Summary](#Summary)
   * [Scikit-learn Pipeline](#Scikit-learn-Pipeline)
   * [AutoML](#AutoML)
   * [Pipeline comparison](#Pipeline-comparison)
   * [References](#References)

***
## Overview
This project is part of the Udacity Azure ML Nanodegree.
In the begining i have created workspece, workspce info detailed above

Workspace name: quick-starts-ws-131112
Azure region: southcentralus
Subscription id: a24a24d5-8d87-4c8a-99b6-91ed2d2df51f
Resource group: aml-quickstarts-131112
Workspace.create(name='quick-starts-ws-131112', subscription_id='a24a24d5-8d87-4c8a-99b6-91ed2d2df51f', resource_group='aml-quickstarts-131112')

Then created cluster, this is the second majority step after creating workspace,there was used recomended "Standard_D2_V2"

from azureml.core.compute import ComputeTarget, AmlCompute
from azureml.core.compute_target import ComputeTargetException

cpu_cluster_name = "compute-cluster"

try:
    compute_target = ComputeTarget(workspace=ws, name=cpu_cluster_name)
    print('Found existing cluster, use it.')
except ComputeTargetException:
    print('Creating a new compute cluster...')
    compute_config = AmlCompute.provisioning_configuration(vm_size='STANDARD_D2_V2', max_nodes=4)
    compute_target = ComputeTarget.create(ws, cpu_cluster_name, compute_config)


Then,in this project  I had the opportunity to build and optimize an **Azure ML pipeline** using the **Python SDK** and a custom **Scikit-learn Logistic Regression** model. I optimised the hyperparameters of this model using **HyperDrive**. Then, I used **Azure AutoML** to find an optimal model using the same dataset, so that I can **compare** the results of the two methods.

Below you can see an image illustrating the main steps I followed during the project.

![Diagram](img/Diagram.JPG?raw=true "Main Steps of the Project")

_Step 1_: Set up the [train script](train.py), create a Tabular Dataset from this [set](https://automlsamplenotebookdata.blob.core.windows.net/automl-sample-notebook-data/bankmarketing_train.csv) & evaluate it with the custom-code Scikit-learn logistic regression model.

_Step 2_:  Creation of a [Jupyter Notebook](udacity-project.ipynb) and use of HyperDrive to find the best hyperparameters for the logistic regression model.

_Step 3_:  Next, load the same [dataset](https://automlsamplenotebookdata.blob.core.windows.net/automl-sample-notebook-data/bankmarketing_train.csv) in the Notebook with TabularDatasetFactory and use AutoML to find another optimized model.

_Step 4_:  Finally, compare the results of the two methods and write a research report i.e. this Readme file. 

***
## Summary
Project's dataset contains marketing data about individuals. The data is related with direct marketing campaigns (phone calls) of a Portuguese banking institution. The classification goal is to predict whether the client will subscribe a bank term deposit (column y).

The best performing model was the **HyperDrive model** with ID HD_fda34223-a94c-456b-8bf7-52e84aa1d17e_14. It derived from a Scikit-learn pipeline and had an accuracy of **0.91760**. In contrast, for the **AutoML model** with ID AutoML_ee4a685e-34f2-4031-a4f9-fe96ff33836c_13, the accuracy was **0.91618** and the algorithm used was VotingEnsemble.

## Scikit-learn Pipeline
**Explain the pipeline architecture, including data, hyperparameter tuning, and classification algorithm.**

**Parameter sampler**

I specified the parameter sampler as such:

```
ps = RandomParameterSampling(
    {
        '--C' : choice(0.001,0.01,0.1,1,10,20,50,100,200,500,1000),
        '--max_iter': choice(50,100,200,300)
    }
)
```

I chose discrete values with _choice_ for both parameters, _C_ and _max_iter_.

_C_ is the Regularization while _max_iter_ is the maximum number of iterations.

_RandomParameterSampling_ is one of the choices available for the sampler and I chose it because it is the faster and supports early termination of low-performance runs. If budget is not an issue, we could use _GridParameterSampling_ to exhaustively search over the search space or _BayesianParameterSampling_ to explore the hyperparameter space. 

**Early stopping policy**

An early stopping policy is used to automatically terminate poorly performing runs thus improving computational efficiency. I chose the _BanditPolicy_ which I specified as follows:
```
policy = BanditPolicy(evaluation_interval=2, slack_factor=0.1)
```
_evaluation_interval_: This is optional and represents the frequency for applying the policy. Each time the training script logs the primary metric counts as one interval.

_slack_factor_: The amount of slack allowed with respect to the best performing training run. This factor specifies the slack as a ratio.

Any run that doesn't fall within the slack factor or slack amount of the evaluation metric with respect to the best performing run will be terminated. This means that with this policy, the best performing runs will execute until they finish and this is the reason I chose it.


## AutoML
**Model and hyperparameters generated by AutoML.**

I defined the following configuration for the AutoML run:

```
automl_config = AutoMLConfig(
    compute_target = compute_target,
    experiment_timeout_minutes=15,
    task='classification',
    primary_metric='accuracy',
    training_data=ds,
    label_column_name='y',
    enable_onnx_compatible_models=True,
    n_cross_validations=2)
```
_experiment_timeout_minutes=15_

This is an exit criterion and is used to define how long, in minutes, the experiment should continue to run. To help avoid experiment time out failures, I used the minimum of 15 minutes.

_task='classification'_

This defines the experiment type which in this case is classification.

_primary_metric='accuracy'_

I chose accuracy as the primary metric.

_enable_onnx_compatible_models=True_

I chose to enable enforcing the ONNX-compatible models. Open Neural Network Exchange (ONNX) is an open standard created from Microsoft and a community of partners for representing machine learning models. More info [here](https://docs.microsoft.com/en-us/azure/machine-learning/concept-onnx).

_n_cross_validations=2_

This parameter sets how many cross validations to perform, based on the same number of folds (number of subsets). As one cross-validation could result in overfit, in my code I chose 2 folds for cross-validation; thus the metrics are calculated with the average of the 2 validation metrics.

## One or more sentences describing the model and parameters generated by AutoML:

AutoML Pipeline: Classification technique used: Multiple Alogritims Best Run Selection: Selected out of multiple runs with diffrent algorititms with there auto generated hyperparameters. Data Pre-Prossesing: The data are cleaned with the clean_data() imported from train.py, with rows with missing values dropped and categorical(textual) fields converted to numerical fields. Confugration Selected: - experiment_timeout_minutes=30 task="classification" enable_early_stopping = True (Enable early termination if the score is not improving in the short term) primary_metric= "AUC_weighted" ( imbalanced data. For example, the AUC_weighted is a primary metric) training_data= train_data (Percentage of labeled data as test data auto selected, we have an option to specify this as well in code) label_column_name="y" (Collumn to be classified) enable_onnx_compatible_models=True (to emable saving output model in ONNX format) n_cross_validations= 3 With the above selected confugration the AutoML Pipeline shows best results with VisualEssemble as AUC Weight (.949 aproximate) being Primary Metric with accuracy of .91 (aproximate).
***
## Pipeline comparison
**Comparison of the two models and their performance. Differences in accuracy & architecture - comments**


| HyperDrive Model | |
| :---: | :---: |
| id | HD_fda34223-a94c-456b-8bf7-52e84aa1d17e_14 |
| Accuracy | 0.9176024279210926 |


| AutoML Model | |
| :---: | :---: |
| id | AutoML_ee4a685e-34f2-4031-a4f9-fe96ff33836c_13 |
| Accuracy | 0.916176024279211 |
| AUC_weighted | 0.9469939634729121 |
| Algortithm | VotingEnsemble |


Two model's reslusts are pretty good. Both methods is comfortable but i thisk Auto Ml is giving more interactive details.The difference in accuracy between the two models is rather trivial and although the HyperDrive model performed better in terms of accuracy, I am of the opinion that the AutoML model is actually better because of its **AUC_weighted** metric which equals to **0.9469939634729121** and is more fit for the highly imbalanced data that we have here. If we were given more time to run the AutoML, the resulting model would certainly be much more better. And the best thing is that AutoML would make all the necessary calculations, trainings, validations, etc. without the need for us to do anything. This is the difference with the Scikit-learn Logistic Regression pipeline, in which we have to make any adjustments, changes, etc. by ourselves and come to a final model after many trials & errors. 

***

## References

- Udacity Nanodegree material
- [Tutorial: Create a classification model with automated ML in Azure Machine Learning](https://docs.microsoft.com/en-us/azure/machine-learning/tutorial-first-experiment-automated-ml)
- [How to Set Up an AutoML Process: Example Step-by-Step Instructions](https://spr.com/how-to-set-up-an-automl-process-example-step-by-step-instructions/)
- Bank Marketing, UCI Dataset: [Original source of data](https://www.kaggle.com/henriqueyamahata/bank-marketing)
- [Create and run machine learning pipelines with Azure Machine Learning SDK](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-create-your-first-pipeline)
- [A Simple Guide to Scikit-learn Pipelines](https://medium.com/vickdata/a-simple-guide-to-scikit-learn-pipelines-4ac0d974bdcf)
- [Building and optimizing pipelines in Scikit-Learn (Tutorial)](https://iaml.it/blog/optimizing-sklearn-pipelines)
- [I had no idea how to build a Machine Learning Pipeline. But here’s what I figured.](https://towardsdatascience.com/i-had-no-idea-how-to-build-a-machine-learning-pipeline-but-heres-what-i-figured-f3a7773513a)
- [Using Azure Machine Learning for Hyperparameter Optimization](https://dev.to/azure/using-azure-machine-learning-for-hyperparameter-optimization-3kgj)
- [hyperdrive Package](https://docs.microsoft.com/en-us/python/api/azureml-train-core/azureml.train.hyperdrive?view=azure-ml-py)
- [Tune hyperparameters for your model with Azure Machine Learning](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-tune-hyperparameters)
- [Configure data splits and cross-validation in automated machine learning](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-cross-validation-data-splits)
- [train-hyperparameter-tune-deploy-with-sklearn](https://githubz.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/ml-frameworks/scikit-learn/train-hyperparameter-tune-deploy-with-sklearn)
- [classification-bank-marketing-all-features](https://githubz.com/Azure/MachineLearningNotebooks/tree/master/how-to-use-azureml/automated-machine-learning/classification-bank-marketing-all-features)
- [Resolving issue with Azure SDK version 1.19](https://knowledge.udacity.com/questions/406220)
- Prevent overfitting and imbalanced data with automated machine learning - [Handle imbalanced data](https://docs.microsoft.com/en-us/azure/machine-learning/concept-manage-ml-pitfalls#handle-imbalanced-data)
- [10 Techniques to deal with Imbalanced Classes in Machine Learning](https://www.analyticsvidhya.com/blog/2020/07/10-techniques-to-deal-with-class-imbalance-in-machine-learning/)


