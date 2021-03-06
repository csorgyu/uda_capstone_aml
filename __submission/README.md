# Heart failure prediction with Azure ML Workspace toolkit
 
The goal of the project is creating a classification model with the 2 primary toolkits Azure ML Workspace offers for a selected dataset.
For my own work I chose the [Heart failure clinical data|https://www.kaggle.com/andrewmvd/heart-failure-clinical-data].

There are 2 major modeling tasks and with deployment tasks included to be delivered and one elective
- [x] Delivering a model with the Auto ML functionality (2 deployment formats, pkl and onnx based model)
- [x] Delivering a model created as my own work and optimized with the hyperdrive feature of Azure ML (3 hyperdrive runs, 2 models, 2 different parameter space sampling on the second)
- [x] Saving the model in ONNX format 

The best model needs to be deployed and tested for consumption in both of the cases.

As a preparation for the project I have created a baseline model on my own computer, to manage my expectations about the prediction excercise.
As the dataset is not huge (300 observations in total), this was easily managable on a single worsktation. From this perspective the dataset does not set challenges in dataset sizing, sampling, management perspective. From the other side it can be easily attached to the git repository as it is the case in this project.
More on the dataset can be read in Dataset overview

During baselining I was experimenting with Random Forest and XGBoost models. Random Forest is a good, well interpretable, robust model for non-linear problems, scaling does not impact the model much.
XGBoost is currently a widely used model, can de used with linear base models and tree based too, I was opting for the second. It iterativerly corrects weak predictors through boosting rounds.
I created a hyperdrive configuration for both of these models, and eventually the AutoML best model was an ensemble built on these type of models.


## Project Set Up and Installation
The project assumes the usage of Azure ML Workspace. I was using the lab environment provisioned by udacity, but it does not have any specific settings, that assumes that environment - in fact I was provided different environment in all of the cases, where the dataset and the notebooks, python scripts were usable.


![image](https://user-images.githubusercontent.com/81808810/119731957-dfcbe780-be77-11eb-9a69-00ffe25b120b.png)

Azre ML workspace provisioning can be done multiple ways, manually from the portal, with ARM template deloyment and from terraform, and it includes several components
* A storage device that serves as a backend for storing data, model reszlts and execution logs
* Azure ML Workspace PaaS service itself
* A keyvault, where secrets and certificates can be stored
* Azure Container Registry for model deployment as a service
* Application Insights for monitoring

The workspace itself uses spark cluster under the hood for compute and inference clusters.

The workspace can be accessed through various API versions, I was personally using the python SDK.

To reproduce my work, the dataset from the ../data folder needs to be registered as a dataset and 4 files need to be uploaded to the the worspace, attached to a compute instance in the following structure:
root
- *hyperparameter_tuning.ipynb*
- *automl.ipynb*
- *scoring_file_v_2_0_0.py*
- *scoring_file_v_2_0_0_onnx.py*
- **SCRIPT**
  - *hyper_tuning_rf.py*
  - *hyper_tuning_xgb.py*

I registered the dataset manually as a tabular dataset, I used the default storage as a datastore. Other datastores can be registered of various types.

The *scoring_file_v_2_0_0.py* and the *scoring_file_v_2_0_0_onnx.py* were collected from the model outputs of the best automl model, and I editet the *_onnx* postfixed one to refer to the .onnx model output.

I was using a 4 node compute cluster for both Auto ML and Hyperdrive based training and ACI based service deployment.

## Dataset

### Dataset overview ###
I downloaded the data from the kaggle link provided above and registered it to the workspace. This could be done through code, however that call is slow, so I wanted to focus lab time on AutoML runs and hyperdrive runs.

The dataset itself contains 13 columns, that describe clinical conditions of 300 observed patients and whether they died or not.
The conditions include:
* age: the age of the patient 
* anaemia: decrease of the red blood cells in the hemoglobin (encoded by 0 and 1)
* creatinine_phosphokinase: level of CPK enzyme in the blood (mcg/L, ranges between 23 and 7861)
* diabetes: if the patient has diabetes (encoded by 0 and 1)
* ejection_fraction: the percentage of blood leaving the heart at each contraction (percentage, ranges between 14 and 80) 
* high_blood_pressure: if patient has hypertension (encoded by 0 and 1 )
* platelets: Platelets in the blood (kiloplatelets/mL, ranges between 25,100 and 850,000)
* serum_creatinine: Level of serum creatinine in the blood (mg/dL, ranges between 0.5 and 9.4)
* serum_sodium: Level of serum sodium in the blood (mEq/L, ranges between 113 and 148)
* sex: woman or man (binary, encoded by 0 and 1)
* smoking: If the patient smokes or not (boolean, encoded by 0 and 1)
* time: follow up periods (days, ranges from 4 to 285)

As we can see, the predictors are all numeric, however the ranges, shown above and the distributions shwn in the 2 examples below suggest, that the predictors are not on the same scale and do not show similar distribution at all, so the problem is really a non-linear one by nature.


![image](https://user-images.githubusercontent.com/81808810/119728522-df315200-be73-11eb-84e2-3912c3951113.png)

**Age distribution example**
The age distribution shows, that the patients, who died were slightly older on average compared to those who survived.



![image](https://user-images.githubusercontent.com/81808810/119729924-7b0f8d80-be75-11eb-96e7-743e25c66bfc.png)

**Distribution of the CPK enzyme levels example**
The distribution of the CPK enzyme is really skewed in both of the cases, mean, Q3 quartile are very similar. The patients who survived show slightly more outliers, but the skewness of the CPK enzyme levels for those who died, is bigger.



### Dataset on portal
![image](https://user-images.githubusercontent.com/81808810/119352400-ef91d300-bca1-11eb-81a3-4a1eaf3ae9e6.png)
The portal shows that after the manual registration the dataset is available after manual registration.

### Data access ccess
I am using a programmatic data access from the notebook side. I was facing network slowness and problems through my modeling, so I decided not to waste time with programmatic registration at each an every lab attempt, just register the downloaded dataset manually.

#### Hyperdrive access
![image](https://user-images.githubusercontent.com/81808810/119352531-16500980-bca2-11eb-9f67-101d43b88b0d.png)
Example shows hyperdrive script based access. The code either checks the registered datasets for a given key, or pulls the data from an URL to register it with a given name. In case of hyperdrive I am using that from the python scripts.

#### Auto ML code access
![image](https://user-images.githubusercontent.com/81808810/120069678-715d7400-c087-11eb-9ecf-cb52c28c4169.png)
In the case of the Auto ML, I am checking the dataset from the ipynb  notebook, and adding the content to a variable, which I pass over to the Auto ML runs.

## Automated ML
In case of Automated ML I am letting the AutoML functionality to run experiments with an initial configuration I set and optimize the models along a primary metric. These runs are registered and the outputs are available on the storage device associated to the workspace.

### AutoML settings and config
![image](https://user-images.githubusercontent.com/81808810/120069647-525ee200-c087-11eb-9b40-b6b3b4398b52.png)
The screenshot above shows the AutoML settings and config I used for the experiment.

The 3 settings I am controlling in the settings are:
* experiment_timeout_minutes: This is a control on the resources I want to allocate to the automated ml experimentations
* max_concurrent_iterations: As we have 4 compute nodes, this is the minimum value that makes sense if I want to be time efficient and maximum value at the same time, as I need to comply with the lab limits
* primary_metric: The different runs of AutoML are optimized by this metric and also compared with each other. There is a set of metrics I can provide, I am optimizing along AUC weighted

The additional AutoML configs are the following
* compute_target: Here I am assigning a compute cluster reference to the AutoML I created formerly (called "auto-ml" in my notebook with 4 nodes)
* task: this is a classification task, in other cases I could opt for regression or timeseries analysis, too
* training_data: this is the data I let the models to train on. If I do not specify validation_data, but also do not specify n_cross_validations, default value is used for csorr validation based on the ![documentation](https://docs.microsoft.com/en-us/azure/machine-learning/how-to-configure-cross-validation-data-splits) . For datasets smaller than 20,000 rows cross validation is used. For datasets smaller than 1000 rows there is 10 fold cross-validation is used, for larger datasets 3 fold. In my case the dataset is very small, so 10 fold will be used. This is good for my purpose.
* label_column_name: this is the target of the predictions, all the rest of the columns will be predictors
* path: project folder TODO fix this later
* enable_early_stopping: if the score is not improving in short term, the further experimentation will stop. Default setting is **False**, so I need to control the behavior
* featurization: My setting is **auto**. This will apply the following automatic featurizations below. As I happen to have numeric features without missing values, this specific setting does not do any improvement or change
  * Categorical: Target encoding, one hot encoding, drop high cardinality categories, impute missing values.
  * Numeric: Impute missing values, cluster distance, weight of evidence.
  * DateTime: Several features such as day, seconds, minutes, hours etc.
  * Text: Bag of words, pre-trained Word embedding, text target encoding.
* debug_log: the file where the debug information will flow
* model_explainability: Whether to enable explaining the best AutoML model at the end of all AutoML training iterations. The default is **True**, but I have emphasized this setting in the config, as I am reflecting on explanaions later
* enable_onnx_compatible_models: Whether to enable or disable enforcing the ONNX-compatible models. The default is *False*, but I want ONNX compatilble models, as that is an exra requirement of the project, so my setting is **True**  

### AutoML Run
![image](https://user-images.githubusercontent.com/81808810/119652654-aae37480-be26-11eb-8e3a-bd7c3f3f0910.png)

#### Data guardrails - 10 fold cross validation proof
![image](https://user-images.githubusercontent.com/81808810/120070487-14fc5380-c08b-11eb-8dcf-cf9ad8b87990.png)
The output of the automl run shows, that the 10-fold cross-validation has actually been applied

#### Data guardrails - Class balancing, missing features and high cardinality check
![image](https://user-images.githubusercontent.com/81808810/120070530-4bd26980-c08b-11eb-924a-24c0c30c48cd.png)
In this dataset, as I formerly mentioned based on the EDA, there are no issing values and the outcomes are balanced. I personally have not performed high cardinality checks, but this AutoML run has done that favor for me

#### Iterations
![image](https://user-images.githubusercontent.com/81808810/120070578-86d49d00-c08b-11eb-97f3-d45c0c6c2cf5.png)

![image](https://user-images.githubusercontent.com/81808810/120070591-95bb4f80-c08b-11eb-98d6-9f7d70712b5c.png)
The 2 images above show, that AutoML has run 51 iterations, and tbe best AUC score was obtained by a VotingEnsamble model. Enabling voting ensembles is an additional feature of AutoML config, the default value is **True** I used this one. These models are ensembles over ensembles, so in certain business cases these may not be allowed, because interpretability may be not straightforward enough.


### Results
*TODO*: What are the results you got with your automated ML model? What were the parameters of the model? How could you have improved it?
#### Model details
![image](https://user-images.githubusercontent.com/81808810/120104816-a389d700-c156-11eb-99fd-2356d7c0acd0.png)
The image above shows the Raw JSON details of the best model, which was a votin ensemble. The algorythm has chosen the AutoML's former runs, to build a voting model, with biggest weight on the 7th, a RandomForest model and additional tree based models, but also a gradient boositng and XGBoost models

![image](https://user-images.githubusercontent.com/81808810/120104946-31fe5880-c157-11eb-8d7f-98f939927920.png)
Interestingly enough some members of the voting ensemble have not reached extremely high AUC score, however what I am assuming here, that the robustness of the random forest algorythm compensates the recall power of xgboost and votes as a regularization, which is important if we observe such a small dataset

#### Metrics of the best run observed on the portal
![image](https://user-images.githubusercontent.com/81808810/119652785-d49c9b80-be26-11eb-801b-8d3024ed13c1.png)
The chart above shows the precision-recall graph. The final model is fairly strong (0.92 weighted AUC), and this specific chart shows, that the precision score (the proportion of true positive predictions from all positives) starts declining only at high recall rates (recall is the proportion of identified positive cases of all positives). This model serves healthcare and particularly it is predicting on potentially lethal outcomes, so the high recall rate is extremely important, however the high precision is also needed, as that false positives are buden on the healthcare system, and nonetheless it is an additional stress on the patients.
 
#### Explanations
![image](https://user-images.githubusercontent.com/81808810/119652882-f39b2d80-be26-11eb-99e4-f169fa528a9d.png)
We can find explanations of certain runs, including the best run on the portal (if not used from the widget view, details below).
This particular chart shows the predictor importance, out of which time was the most important and the ejection fraction and serum creatine were the most important. This means, that followup time with the patient had key importance and the proportion of blood leaving the heart is one of the key aspects medical staff needs to focus on as well as the creatine level in the blood.

![image](https://user-images.githubusercontent.com/81808810/120071014-850bd900-c08d-11eb-850e-ea400181b899.png)


#### Conda environment
![image](https://user-images.githubusercontent.com/81808810/120070858-ccde3080-c08c-11eb-8fb9-42e53db075ff.png)
For reproducibility, the AutoML registers the environment dependencies it ran with.
This is used further when we are deploying a model, because we need to pass the environment to the running container.

#### Confusion matrix
![image](https://user-images.githubusercontent.com/81808810/120070823-a28c7300-c08c-11eb-96bc-32bfe4f00674.png)
Can be obtained from the run details, from the backend. Probably more useful if queried programmatically or used from the widget view, shown below. Neverteheless the information can be made available for any other visualization or postprocessing tool accessing the storage (PowerBI, R users,...)

#### Progress - Widget
![image](https://user-images.githubusercontent.com/81808810/119653166-4c6ac600-be27-11eb-8e51-14127ecc9c87.png)

One can followup the progress on the RunDetails widget. It helps with progress, whather certain runs have failed or not, the current iterations metric, theformer best metric, the duration, start and end time. It also shows the pipeline, that was created automatically. After finalizing it shows the best metrics scores (in my case AUC_weighted) on a diagram.

#### Results: Charts
![image](https://user-images.githubusercontent.com/81808810/120074651-17b47400-c09e-11eb-9187-458a54d74f6e.png)

ROC-precision curve available on the widget, no need to go to the actual run details on the poral

![image](https://user-images.githubusercontent.com/81808810/120074679-3dda1400-c09e-11eb-9ccb-4d252ed87f84.png)

The confusion matrix is available in nice visual format in the widget view, without leaving the notebook experience. In case portal based wrangling is disfavored and the storage access directly is disfavored, this is the best option to get immediate information from ML developer side.

![image](https://user-images.githubusercontent.com/81808810/120074693-51857a80-c09e-11eb-8f52-26846662e1c4.png)

Feature importance is shown, also these are the results of featurization



#### Results: Transformations - Widget
![image](https://user-images.githubusercontent.com/81808810/120074605-eb005c80-c09d-11eb-8b6d-5aa8e1f9b14e.png)

Transformation graph can be checked directly from the widget

#### Metrics
![image](https://user-images.githubusercontent.com/81808810/119653970-3c071b00-be28-11eb-98da-242b71847de5.png)

The screenshot above shows that all metrics can be retrieved, not just the primary one, so recall, F1-score, accuracy, precision and other scores too.
There is a reference for the confusion matrix and the accuracy table, these can be obtained from the blob storage associated to the ML Workspace

#### Features
![image](https://user-images.githubusercontent.com/81808810/120109539-1f414f00-c16a-11eb-8072-f9ef5820a624.png)

The best model run shows, that only 7 colums were used as numeric, the rest were actually used as categorical in the automatic featurization

#### ONNX model
![image](https://user-images.githubusercontent.com/81808810/120070773-6a853000-c08c-11eb-8f22-2329a34b5cd8.png)
We have the onnx model version generated too in the output folder, on the top of the regular model.pkl file

#### Saving ONNX model
![image](https://user-images.githubusercontent.com/81808810/120071523-1bd99500-c090-11eb-811c-d55475fd2645.png)

I am simply saving the model in the project directory. Later on this could be moved and deployed to any other environment.
#### Registering PKL and ONNX models both
![image](https://user-images.githubusercontent.com/81808810/120105542-9fab8400-c159-11eb-83e5-e3ca9f485ce8.png)

I created an ONNX model registration in the model repo and a pkl version as well. The conda environment was retreived from the run itself, to make all configs and settings reproducible.

#### Inference config
![image](https://user-images.githubusercontent.com/81808810/120072224-02861800-c093-11eb-9826-ac3ab7e5e918.png)

I collected the scoring files from the model output.

![image](https://user-images.githubusercontent.com/81808810/120105501-77238a00-c159-11eb-988f-d564169a94e9.png)

Uploaded them to the project repo.

#### Deploying PKL model with inference config
![image](https://user-images.githubusercontent.com/81808810/120105581-d08bb900-c159-11eb-9143-d8926b72f998.png)

Using the scoring file and the environment that I retreived in the steps above, I created a ACI config, where I specified the allocated CPU, memory, whether I want to enable authentication and whether I want to enable app insight.

#### Deployment status check on portal
![image](https://user-images.githubusercontent.com/81808810/120107551-b5bd4280-c161-11eb-93b8-1a251ed4b933.png)

The portal view and the code call result both show, that the deployment is successful.

#### Deploying ONNX model with inference xonfig
![image](https://user-images.githubusercontent.com/81808810/120106067-dd111100-c15b-11eb-951f-95f4170302a2.png)

Specifying onnx related settings, but keeping the same ACI config, I deployed the ONNX model version, too

#### Experiment final details
![image](https://user-images.githubusercontent.com/81808810/120109465-e012fe00-c169-11eb-8e19-515f3a7ec929.png)

We can see the model result, AUC, registered models with versions and deployed models with versions on the experiment summary



### Model usage details in swagger
![image](https://user-images.githubusercontent.com/81808810/120108966-bc4eb880-c167-11eb-9ef1-416485b1bc84.png)

If we download swagger.json file from the deployed model and set up a local container for swagger and a HTTP client, we can see an example POST request payload, what we can sent to the model.

![image](https://user-images.githubusercontent.com/81808810/120109023-fa4bdc80-c167-11eb-8e26-bb840612843f.png)

We can expect a single prediction number as a response. 

### Using Auto ML Model
![image](https://user-images.githubusercontent.com/81808810/120107603-f4eb9380-c161-11eb-82ba-b2d0bd9a9223.png)

The code based call provides the prediction for us, no errors shown. I tested with multiple payloads, both 0 and 1 predictions have been returned.

### Logs
![image](https://user-images.githubusercontent.com/81808810/120107640-1ea4ba80-c162-11eb-8ac8-897d18644029.png)

Logs show healthy enpoint request-response communication, no error was thrown during the prediction.




## Hyperparameter Tuning
*TODO*: What kind of model did you choose for this experiment and why? Give an overview of the types of parameters and their ranges used for the hyperparameter search

I was experimenting on 3 layers:
* First I checked the performance of 2 model types: Random Forest and XGBOOST
* Second, I ran a random parameter sampling based hyperparameter run with the better model (XGBOOST)
* Third, I checked what parameters I assumed have less impact on the model, and have run a Bayesian parameter sampling based hyperdrive run on the rest of the parameters
* As an outcome, my best model was unluckily performing worse still, than the AutoML
* Within the scope of tha project I was focusing on exploration of the techniques, so after this 3 layered approach iI gave up optimizing and opted for the best AutoML model as an end result

![image](https://user-images.githubusercontent.com/81808810/119644203-e8430480-be1c-11eb-9950-7d348d5c316d.png)

The screenshot above shows the multiple runs of hyperdrive experimentations.
The sections below contain the datails.

### Model01: The Random Forest
The first modeling excercise was running a random forrest classifier with different parameters.
The parameters I was trying to control were:
* n_estimators: The number of trees used in the random forest
* max_depth: the maximum depth of the trees
* min_samples_split: the minimum number of samples to continue splitting

#### Parameter space
![image](https://user-images.githubusercontent.com/81808810/120106889-1434f180-c15f-11eb-87e3-83104c1c5995.png)

The picture above shows the parameter space I went through with random parameter sampling 

#### Training progress
##### Portal view
![image](https://user-images.githubusercontent.com/81808810/119352067-827e3d80-bca1-11eb-8580-3fe832ac8ca0.png)

##### Widget view
The vidget views show the progress as well as the model performance growth after the given runs.

![image](https://user-images.githubusercontent.com/81808810/119352190-a93c7400-bca1-11eb-8022-167a105efb70.png)

![image](https://user-images.githubusercontent.com/81808810/120109309-26b42880-c169-11eb-91f1-14081ce0d8f1.png)

#### Best metric for Random Forest
![image](https://user-images.githubusercontent.com/81808810/120109891-b1962280-c16b-11eb-944e-b0e3bd991901.png)
I have obtained 0.796 AUC with the Random Forest classifier. This is not bad, but not perfect either, especially if we are talking about a model deciding on life saving help. SO I use this result as a baseline.

### Model02: XGB with random parameter search
#### Parameter space and sampling
![image](https://user-images.githubusercontent.com/81808810/120106934-434b6300-c15f-11eb-8e4a-1dc8e1cff36c.png)

For XGBoost I used the following parameters duing the tuning
* num:boost_round: the number of boosting rounds to optimize the weak estimator
* max_depth: I am using tree based learners, and this parameter controls the maximum depth. Maximum depth of a tree. Increasing this value will make the model more complex and more likely to overfit. Default is 6.
* learning_rate: Step size shrinkage used in update to prevents overfitting. After each boosting step, we can directly get the weights of new features, and eta shrinks the feature weights to make the boosting process more conservative.
* gamma: Minimum loss reduction required to make a further partition on a leaf node of the tree. The larger gamma is, the more conservative the algorithm will be
* reg_lambda: L2 regularization term on weights. Increasing this value will make model more conservative.
* scale_pos_weight: Control the balance of positive and negative weights, useful for unbalanced classes

#### Portal view
![image](https://user-images.githubusercontent.com/81808810/119645118-f9404580-be1d-11eb-9ad6-6f98b5d25bf2.png)

One can opt for following up the progress on the portal. This chart shows the progress of the random forest classifier training, given the actual input values for the parameters.

#### Widget view
![image](https://user-images.githubusercontent.com/81808810/119645070-e88fcf80-be1d-11eb-8aec-13b93e652475.png)

The widget shows progress in the development space in real time.

### Model03: XGBoost with bayesian parameter search
#### Paramter space
![image](https://user-images.githubusercontent.com/81808810/120107510-91616600-c161-11eb-989e-0bb7b6ee31b1.png)

For the second xgboost run I was trying to rule out those parameters, that were not really deciding factors in the previous runs, and used those parameters, that seemed to make difference. I also used different parameter sampling, as I wanted more exhaustive search in the smaller space.
Here I am controlling:
* num:boost_round: the number of boosting rounds to optimize the weak estimator
* max_depth: I am using tree based learners, and this parameter controls the maximum depth
* learning_rate: The learning rate of the process
* I keep gamma on 2, lambda on 5 and scale_pos_weigth on 1, as the classes are balanced

#### Training process
![image](https://user-images.githubusercontent.com/81808810/120109703-d938bb00-c16a-11eb-8a52-da316002ef6d.png)

The portal view shows how the AUC score of the given runs change.

### Best metrics for hyperdrive experiments
![image](https://user-images.githubusercontent.com/81808810/120110134-a263a480-c16c-11eb-86b1-6ede81b6ff86.png)

The best xgboost model ended up performing similarly, slightly worse than the RF. This is in gerneral suprising, but auto ML also ended up wit using more random forest alike classifiers in the voting ensamble

### Registered model at the end of hyperdrive experimentation
![image](https://user-images.githubusercontent.com/81808810/120110218-e22a8c00-c16c-11eb-9023-54836a00c24f.png)

I registered the random frest ultimately. This model I have deployed from the portal, manually.

### Deployed model
![image](https://user-images.githubusercontent.com/81808810/120111133-00928680-c171-11eb-8f48-bfcdfdc7b199.png)

This model I deployed manually, but the application insights I have enabled from code. The endpoint is in a healthy state and the test below was also successful.



### Using model
![image](https://user-images.githubusercontent.com/81808810/120111072-b6a9a080-c170-11eb-86ae-d83141b0d8b9.png)
The usage of the model is slightly different, here the returned values are probabilities. For the same input we see the probability of a negative case and the positive, so overall the prediction made for the same test is the sam in this case as automl endpoint model

### Logs
![image](https://user-images.githubusercontent.com/81808810/120111059-a691c100-c170-11eb-87f1-873d9bf87674.png)

T




## Deleting compute 
![image](https://user-images.githubusercontent.com/81808810/120110929-fd4acb00-c16f-11eb-910c-2f4af13a6d76.png)

Compute has been deleted from notebook.

## SUMMARY
The project has proven that in a timecruch scenario AutoML proved really useful, to find the best model. In fact the AutoML best model was a true surprise, because I was expecting mode xgboost related models in the voting ensemble. This shows, that random forest, which was my offline original choice can be used well. The AutoML has also shown, that featurization need to be done more carefully compared to my bacis usage.

## Screen Recording
*TODO* Provide a link to a screen recording of the project in action. Remember that the screencast should demonstrate:
- A working model
- Demo of the deployed  model
- Demo of a sample request sent to the endpoint and its response

### Auto ML Screecast
https://youtu.be/a1B01lm_AO4

### Hyperdrive screencast01
https://youtu.be/7ev5_N2jOWY

### Hyperdrive screencast02 and compute delete
Making sure, that model response is captured, and the deleting of the compute cluster
https://youtu.be/SOridasbanE

## Standout Suggestions
### Delivered: ONNX model
I saved my model in an ONNX version. This is a opern neural network exchange format, that allows interchange between various ML frameworks and tools. ONNX models can be easily integrated to Windos apps and devices

### Potential upgrade of the model
The model was generated on  small data sample. It would be very interesting, to backtest the model in true batch inference case. This for sure cannot be done in the scope of the project, however that gives the ML models a real usage context - and value.


## Final notes
![image](https://user-images.githubusercontent.com/81808810/120106185-6c1e2900-c15c-11eb-9838-1e80d59ec588.png)

Netwok errors were continuously present in the lab environment in this lab, that made the notebook view very challenging to use.
