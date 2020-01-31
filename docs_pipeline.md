# AIML Disease Detection Pipeline - A Data Scientist's Primer

## Starting on a new disease

### Dolphin and Hadoop setup on CDSW

Please refer to the [Confluence page](https://jiraims.rm.imshealth.com/wiki/display/PE/Running+Dolphin+on+CDSW)
for instructions on setting up a new project on CDSW. Do note the following:

- You only need to follow the instructions in the first two sections: "Deploy Dolphin on CDSW" and "From CDSW interface" to create the project. Oher subsections are referred to below.
- CDSW login details are the same as your IQVIA login details
- However, Hadoop authentication uses different credentials. The username is provided
on the [Confluence page](https://jiraims.rm.imshealth.com/wiki/display/PE/Running+Dolphin+on+CDSW). For the password,
please ask a data scientist or a big data engineer in the team stating that you require the password for the `rwipusr` subtenant.

### Folder structure for a new disease

It is advisable to keep files for each disease organised separately. Note that, by default, the home directory on CDSW is located at `/home/cdsw/`.

The following is a suggested structure that could be used for a new disease (in this case the name of the disease is "hfpef") relative to the home directory:

- `configs/hfpef/`: For Dolphin configuration files
- `modelscripts/hfpef/`: For Python scripts used to analyse and preprocess the data etc.
- `data/hfpef/`:
  - `data/hfpef/original/`: For original PosNeg and Scoring data as provided by ETL
    - `data/hfpef/original/posneg_v1/`
    - `data/hfpef/original/scoring_v1/`
  - `data/hfpef/preprocessed/`: For preprocessed data to be consumed by Dolphin
    - `data/hfpef/preprocessed/posneg_v1/`
    - `data/hfpef/preprocessed/scoring_v1/`
  - `data/hfpef/output_of_some_model/`: For Dolphin output. NB: You do *not* need to actually create this; it will, automatically be created based on the settings in the configuration file (`data.directories.output` parameter)
  - `data/hfpef/predictions/`: For prediction files generated from Dolphin output
    - `data/hfpef/predictions/final_prediction_files`

NB: Above assumes that we are using version 1 of the cohorts. If other versions are used, adjust the paths accordingly.

## Data acquisition

The Data Scientist's involvement with the Disease Detection pipeline typically starts when the PosNeg and Scoring Cohorts are prepared by the ETL team and placed on the DRML server. This will typically be communicated via one of:

- E-mail
- Teams message (e.g. there is a dedicated "Disease Job Running" chat)
- Jira ticket (e.g. the "Platform Operationalisation" board)

The Data Scientist responsible for a particular disease pipeline needs to make sure that they are included in the relevant communication channels.

When ETL inform Data Scientists that the PosNeg and Scoring Cohorts are available on DRML they should provide the exact location of these files. Typically, this data will be found on HDFS at

`/production/rwi/data/mli/output/disease/{DISEASE NAME}/{COHORT NAME}/`

where `{DISEASE NAME}` and `{COHORT NAME}` would be supplied by ETL. Alternatively, it is possible to infer the path by exploring `/production/rwi/data/mli/output/disease/` using `hdfs dfs -ls` command from a CDSW terminal. Note that each cohort folder typically contains multiple csv files.

Files on DRML can also be accessed using a graphical interface (HUE): https://usrhdphue.rxcorp.com:8889/hue/accounts/login/?next=/hue/ (user: `rwipusr`, password: to be provided)

Assuming the folder structure has been created on CDSW and the location of the data has been specified, the files can be copied from HDFS to CDSW using `hdfs dfs -get <source> <target>` command (from a terminal). For example, the following would copy version 2 of HFpEF PosNeg cohort from HDFS to CDSW using the folder structure above:

`hdfs dfs -get /production/rwi/data/mli/output/disease/hfpef/octopus_novartis_generic_hfpef_posneg_v2/*.csv /home/cdsw/data/hfpef/posneg_v2/`

## Data verification

Once the data is obtained, it is advisable to verify it with the product owners and the clinical team. Typically, we confirm:

- the sizes of the cohorts,
- the number of positive cases in the PosNeg cohort,
- the number of features that will be ingested by Dolphin,
- the list of clinical events from which the features were generated.

## Data preparation

The data that is obtained from ETL needs to be preprocessed before it is suitable to be consumed by Dolphin. Currently the following preprocessing is carried out:

- features associated with CC02 events are dropped
- the following columns are also dropped: "lrx_flag", "dx_flag", "lookback_dys",
"lookback_mnths", "lookback_dt", "back_dt", "frst_rx_clm_dt", "frst_dx_clm_dt",
"lst_rx_clm_dt",  "lst_dx_clm_dt", "disease_frst_exp_dt","index_dt"

The Scoring Cohort is typically orders of magnitude larger than the PosNeg cohort and it is not
necessary to preprocess it before the model is run (but it should be done by the time model training is complete). For this reason we typically preprocess PosNeg and Scoring Cohorts separately.

The sample script for preprocessing can be found [on Pythia](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/Pythia/blob/develop/snippets/disease_detection/preprocess_raw_data.py). Two copies, one for each cohort, can be saved in `modelscripts/{DISEASE}/` folder as described above and paths modified as necessary.

## Model training

The model is trained using Dolphin (see [Dolphin User Guide](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/dolphin/blob/develop/doc/user/USER_GUIDE.md) for details). This requires two inputs:

- the preprocessed PosNeg cohort; and
- a configuration file

Examples of configuration files are available. The following typically need to be customised for a new model:

- `[project]` should accurately describe the model
- `[data.data_source_info]` should specify the HDFS path where the PosNeg cohort was obtained and the number of csv files that comprosied the PosNeg data set. The `comment` field should state "HDFS DRML".
- `[data.directories]` should use the preprocessed PosNeg cohort as an input and specify the output path for the model
- `[optimiser.max_evals]` varies. If POs need a proof-of-concept model with a quick turnaround `max_evals = 1` can be used. Otherwise, typical values are 25, 40, 50, 75.
- `[model_params]` is often unchanged but there are two special cases:
  - `[model_params.num_threads]` should reflect the number of vCPUs used by the CDSW session where the model is run
  - If `max_evals = 1` then LightGBM parameters should be specified explicitly to avoid Hyperopt making an arbitrary choice. Thus when using example configuration files from previous projects, take care to use those from other projects with `max_evals = 1`.

The configuration file to be use can be saved `configs/{DISEASE}/` folder as described above and modified as necessary.

## Prediction generation

The ETL pass information about diagnosed and predicted patients (by our model) to UI using a `prediction.csv` file. This file contains data about all the diagnosed patients and all the patients from the scoring cohort, predicted as positive by the model The `prediction.csv` is a file that (currently) contains 5 columns:
- `patient_id`
- `pat_gender`
- `pat_age`
- `index_dt` (index date)
- `label` (`1` indicates a positive patient (from the pos_neg cohort), `0` indicates a patient from the scoring cohort, i.e. not diagnosed)
- `prediction` (`1` indicates a patients predicted as positive by the model, `0` indicates a patient not predicted as positive by the model).

We generate the prediction file in several steps, described below.

### Create predicted probabilities for the Scoring Cohort

Once Dolphin completes, the model will be saved in a `model_{PIPELINE_ID}.txt` file in the output path specified in the configuration file. This can then be used to generate predictions for the Scoring Cohort. Use [the sample script](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/Pythia/blob/develop/snippets/disease_detection/predict_scoring_cohort.py) to generate the predicted probabilities. The sample script should be saved in `modelscripts/{DISEASE}/` folder as described above and manually modified to specify the paths used by the script.

### Decide on a precision target

The predicted probabilities then need to be converted to actual binary prediction. To do that we need to specified a threshold that would separate the predicted probabilities into positive and negative predictions. Depending on the choice of the threshold, the precision of the model will change. We tend to target high precision (~85%) but if that results in a very low or very high number of predicted patients we adjust the precision level. Therefore, we need to set the precision level by consulting with the clinical team before the final prediction file can be produced.

A [sample script](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/Pythia/blob/develop/snippets/disease_detection/thresholds.py) is used to produce information necessary for the clinical team to provide their feedback. A copy can be saved in `modelscripts/{DISEASE}/` folder as described above and manually modified to specify:

- the paths used by the script
- `diagnosed_num` - the number of positive labels in the PosNeg Cohort
- `target_precisions` - the list of precision values that we wish to consider (typically these are 75%,80%, 85%, 90%, 95% and 99%)

When the script is run it will produce, for each target precision level,

- the threshold that would give this precision;
- the number of positive patients in the Scoring Cohort that the model would predict; and
- the total number of positive patients that we obtain between PosNeg and Scoring Cohorts.

The number of positive patients (predicted and overall) and the model performance measured as its average precision should be circulated to the clinical team and the product owner (no need to circulate the thresholds). The clinical team/POs will advise which precision level we should target based on which numbers most closely match the prevalence levels observed in reality.

Once the precision level is fixed, the threshold to be used for the prediction file is then determined.

### Determine diagnosed positives from the PosNeg Cohort

For the PosNeg Cohort we do not need to obtain any predictions. We only need to identify which patients in the cohort have been diagnosed as positive. The sample script for this can be found [here](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/Pythia/blob/develop/snippets/disease_detection/create_prediction_file_positives.py) and can saved in `modelscripts/{DISEASE}/` folder as described above and manually modified to specify the paths.

### Create final prediction file

For the Scoring Cohort, predictions can be obtained once the threshold is set, as above. Then the diagnosed patients from the PosNeg Cohort and the predicted patients from the Scoring Cohort need to be combined into a single prediction file in a specified format. The sample script for this can be found [here](http://rwes-gitlab01.internal.imsglobal.com/Prototypes/Pythia/blob/develop/snippets/disease_detection/create_final_prediction_file.py). The sample script should be saved in `modelscripts/{DISEASE}/` folder as described above and manually modified to specify the paths used by the script.

The prediction file will always be saved as `prediction.csv`. Therefore if multiple models are run on the same input data, separate folders can be created under `data/hfpef/predictions/final_prediction_files` for different models to differentiate these.

### Upload the prediction file

Once the prediction file is created it needs to be uploaded to HDFS using `hdfs dfs -put <source> <target>` command. The typical location for prediction files is `/production/rwi/data/mli/input/disease/{DISEASE NAME}/{MODEL NAME}/prediction.csv`.

`{MODEL NAME}` is arbitrary. We usually assign it a version number. So the first model for the disease is, for example `v1`.

Once the prediction file is uploaded (which can be verified by using `hdfs dfs -ls`) the upload path needs to be circulated to the ETL team.
