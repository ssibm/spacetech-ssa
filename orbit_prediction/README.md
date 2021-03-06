# Introduction

The orbital prediction package explores the possibilities of improving traditional physics based models of orbital dynamics using machine learning techniques.


# Dependencies

This package has been developed and tested on Python 3.8. We recommend using [conda](https://docs.conda.io/en/latest/) or [Python's virtual environments](https://docs.python.org/3/library/venv.html) for keeping dependencies separate from your system Python installation, as an example

```shell
conda create -n ssa python=3.8
conda activate ssa
```

or

```shell
python3 -m venv ssa_env
source ssa_env/bin/activate
```

Then the Python based dependencies can be installed by running

```sh
pip install -r requirements.txt
```

from this directory.


# Data Flow

![img](data_flow.png)


# Pipeline Components


## ETL Orbit Data from USSTRATCOM

We utilize orbit data from [United States Strategic Command](https://en.wikipedia.org/wiki/United_States_Strategic_Command) (USSTRATCOM) via the [space-track.org](https://www.space-track.org/) website and API. In order to access this API, you must register for an account [here](https://www.space-track.org/auth/createAccount). The data is served in the [two-line element set](https://en.wikipedia.org/wiki/Two-line_element_set) format which is a fixed width text format that conatains the [Keplerian orbital elements](https://en.wikipedia.org/wiki/Orbital_elements) of an RSO at a point in time. We then parse the TLE data and calculate the position (**r**) and velocity (**v**) [orbital state vectors](https://en.wikipedia.org/wiki/Orbital_state_vectors).


### ETL CLI

The [ETL module](orbit_prediction/spacetrack_etl.py) provides a CLI with the following arguments

-   `--st_user`: The username for space-track.org
-   `--st_password`: The password for space-track.org
-   `--norad_id_file`: The path to a text file containing a single NORAD ID on each row to fetch orbit data for. If no file is passed then orbit data for all LEO RSOs will be fetched.
-   `--past_n_days`: The number of days into the past to fetch orbit data for each RSO, defaults to 30 days.
-   `--only_latest`: A boolean flag to only fetch the latest TLE for each RSO.
-   `--output_path`: The path to save the orbit data parquet file to.


### Running ETL Script Examples

Running

```sh
python orbit_prediction/spacetrack_etl.py --st_user <SPACE TRAC USERNAME> \
       --st_password <SPACE TRACK PASSOWRD> \
       --norad_id_file sample_data/test_norad_ids.txt \
       --past_n_days 10 \
       --output_path <OUTPUT>
```

will retrieve orbit data from the past 10 days only for the RSOs with the NORAD IDs listed in [this file](sample_data/test_norad_ids.txt).

Running

```sh
python orbit_prediction/spacetrack_etl.py --st_user <SPACE TRAC USERNAME> \
       --st_password <SPACE TRACK PASSOWRD> \
       --only_latest \
       --output_path <OUTPUT>
```

will fetch just the last TLE for every RSO in LEO.


### Note on API Rate Limits

The USSTRATCOM API is throttled and the amount of data that can be in the response of a single query is limited. The [space-track.org API client](https://pypi.org/project/spacetrack/) we use automatically [rate limits](https://spacetrack.readthedocs.io/en/latest/usage.html#rate-limiter) the number of requests however it is a good idea to review the [API guidlines](https://www.space-track.org/documentation#api-use-guidelines) to understand how often you can run automated scripts against the API.


### Resulting Data

The result of running the ETL script is a a [pandas](https://pandas.pydata.org) [DataFrame](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html) that is saved in the [Parquet](https://parquet.apache.org) format with the following columns:

| Field                 | Description                                         | Type     |
|--------------------- |--------------------------------------------------- |-------- |
| rso<sub>id</sub>      | The unique ID for the RSO                           | string   |
| rso<sub>name</sub>    | The name of the RSO                                 | string   |
| epoch                 | The timestamp the orbital observation was taken     | datetime |
| r<sub>x</sub>         | The `x` component of the position vector `r`        | float    |
| r<sub>y</sub>         | The `y` component of the position vector `r`        | float    |
| r<sub>z</sub>         | The `z` component of the position vector `r`        | float    |
| v<sub>x</sub>         | The `x` component of the velocity vector `v`        | float    |
| v<sub>y</sub>         | The `y` component of the velocity vector `v`        | float    |
| v<sub>z</sub>         | The `z` component of the velocity vector `v`        | float    |
| object<sub>type</sub> | Whether the RSO is a paylod, rocket body, or debris | string   |


## Physical Model Orbit Prediction

The [physics model module](orbit_prediction/physics_model.py) uses the [poliastro](https://docs.poliastro.space/en/stable/) astrodynamics library to build a training data set of the predictions and errors made by a physical model so that we can possibly train machine learning models to estimate this prediction error. The baseline physics model is a [two body model](https://en.wikipedia.org/wiki/Two-body_problem) that uses [Cowell's formulation](https://en.wikipedia.org/wiki/Perturbation_(astronomy)#Cowell.27s_formulation) for modeling the perturbation in a RSO's orbit caused by the Earth. We build our training set by:

1.  Given an orbit data point for an RSO, we find all the orbit data points for that RSO that are within `n` days after the given data point.
2.  We then create a physics model starting at the at the given orbit data point and propagate the orbit to all the data points that are within `n` days in the future.
3.  We use the orbit data points as ground truth to determine the error in the physical model's propagation.


### Training Data Creation CLI

The CLI to create a training data set has the following arguments:

-   `--input_path`: The path to the parquet file to load the orbit observations from.
-   `--output_path`: The path to save the prediction/error training dataset to.
-   `--last_n_days`: Only use observations from the last \`n\` days when creating the prediction windows. Defaults to 30.
-   `--n_pred_days`: The number days in the prediction window. Defaults to 7.


### Resulting Data

The result of running the training data creation script has the following columns:

| Field                                          | Description                                                      | Type     |
|---------------------------------------------- |---------------------------------------------------------------- |-------- |
| rso<sub>id</sub>                               | The unique ID for the RSO                                        | string   |
| rso<sub>name</sub>                             | The name of the RSO                                              | string   |
| epoch                                          | The timestamp the orbital observation was taken                  | datetime |
| r<sub>x</sub>                                  | The `x` component of the position vector `r`                     | float    |
| r<sub>y</sub>                                  | The `y` component of the position vector `r`                     | float    |
| r<sub>z</sub>                                  | The `z` component of the position vector `r`                     | float    |
| v<sub>x</sub>                                  | The `x` component of the velocity vector `v`                     | float    |
| v<sub>y</sub>                                  | The `y` component of the velocity vector `v`                     | float    |
| v<sub>z</sub>                                  | The `z` component of the velocity vector `v`                     | float    |
| object<sub>type</sub>                          | Whether the RSO is a paylod, rocket body, or debris              | string   |
| start<sub>epoch</sub>                          | The `epoch` when the prediction was started                      | datetime |
| elapsed<sub>seconds</sub>                      | The number of seconds between the `start_epoch` and `epoch`      | float    |
| physcis<sub>pred</sub><sub>r</sub><sub>x</sub> | The `x` component of the predicted position vector `r`           | float    |
| physcis<sub>pred</sub><sub>r</sub><sub>y</sub> | The `y` component of the predicted position vector `r`           | float    |
| physcis<sub>pred</sub><sub>r</sub><sub>z</sub> | The `z` component of the predicted position vector `r`           | float    |
| physcis<sub>pred</sub><sub>v</sub><sub>x</sub> | The `x` component of the predicted velocity vector `v`           | float    |
| physcis<sub>pred</sub><sub>v</sub><sub>y</sub> | The `y` component of the predicted velocity vector `v`           | float    |
| physcis<sub>pred</sub><sub>v</sub><sub>z</sub> | The `z` component of the predicted velocity vector `v`           | float    |
| physics<sub>err</sub><sub>r</sub><sub>x</sub>  | The prediction error in the `x` component of the position vector | float    |
| physics<sub>err</sub><sub>r</sub><sub>y</sub>  | The prediction error in the `y` component of the position vector | float    |
| physics<sub>err</sub><sub>r</sub><sub>z</sub>  | The prediction error in the `z` component of the position vector | float    |
| physics<sub>err</sub><sub>v</sub><sub>x</sub>  | The prediction error in the `x` component of the velocity vector | float    |
| physics<sub>err</sub><sub>v</sub><sub>y</sub>  | The prediction error in the `y` component of the velocity vector | float    |
| physics<sub>err</sub><sub>v</sub><sub>z</sub>  | The prediction error in the `z` component of the velocity vector | float    |


## Training Machine Learning Models

The [error prediction module](orbit_prediction/pred_physics_err.py) provides a process for using [XGBoost](https://xgboost.readthedocs.io/en/latest/) to build baseline [gradient boosted](https://en.wikipedia.org/wiki/Gradient_boosting) [regression trees](https://en.wikipedia.org/wiki/Decision_tree_learning) models to estimate the error made by the physics model in predicting orbits.


### Features

The features used by the baseline models are:

-   `elapsed_seconds`: The number of seconds that the physical model predicted into the future.
-   `physcis_pred_r_x`: The `x` component of the predicted position vector `r`
-   `physcis_pred_r_y`: The `y` component of the predicted position vector `r`
-   `physcis_pred_r_z`: The `z` component of the predicted position vector `r`
-   `physcis_pred_v_x`: The `x` component of the predicted velocity vector `v`
-   `physcis_pred_v_y`: The `y` component of the predicted velocity vector `v`
-   `physcis_pred_v_z`: The `z` component of the predicted velocity vector `v`


### ML Models

We independently build a baseline [XGBRegressor](https://xgboost.readthedocs.io/en/latest/python/python_api.html#module-xgboost.sklearn) model for each of the six error columns:

-   `physics_err_r_x`
-   `physics_err_r_y`
-   `physics_err_r_z`
-   `physics_err_v_x`
-   `physics_err_v_y`
-   `physics_err_v_z`


### Training CLI

We can train the baseline models by running

```sh
python orbit_prediction/pred_physics_err.py
```

with the following arguments:

-   `--input_path`: The path to the parquet file containing the physical model prediction/error training data.
-   `--use_gpu`: A boolean flag for whether or not to use GPUs in training. Requires CUDA dependencies are properly installed, see [here](https://xgboost.readthedocs.io/en/latest/gpu/index.html) for more details.
-   `--out_dir`: The directory to serialize the JSON representations of the models to.


## Combining Physics and ML Models

Finally we combine the physical orbit model and the machine learning models by altering the physics predicted state vectors by the error amounts predicted by the ML models.


### CLI for Making Final Orbit Predictions

The [orbit prediction module](orbit_prediction/pred_orbits.py) has a CLI to:

1.  Fetch the most up to date orbit data for LEO RSOs from USSTRATCOM.
2.  Use a physical model to predict the future RSO orbits.
3.  Correct the physical model predictions using the errors predicted by the ML models.

The CLI can be run via

```sh
python orbit_prediction/pred_orbits.py
```

with the following arguments:

-   `--st_user`: The username for space-track.org
-   `--st_password`: The password for space-track.org
-   `--norad_id_file`: The path to a text file containing a single NORAD ID on each row to fetch orbit data for. If no file is passed then orbit data for all LEO RSOs will be fetched.
-   `--ml_model_dir`: The path to the directory containing the error prediction models serialized as JSON.
-   `--n_days`: The number of days in the future to make orbit predictions for, defaults to 7.
-   `--timestep`: The frequency in seconds to make orbit predictions for, defaults 600
-   `--output_path` The path to save the orbit prediction pickle file to. These results can be used directly by the [conjunction search UI](../conjunction_search/README.md) to search and visualize what the predicted space traffic will look like.


# Pipeline Demo

Running the [pipeline demo script](pipeline_demo.sh) will run the whole orbital prediction pipeline for the RSOs listed in [this file](sample_data/test_norad_ids.txt). First we need to set the needed environment variables

```shell
export ST_USER=<SPACE TRACK USERNAME>
export ST_PASSWORD=<SPACE TRACK PASSWORD>
```

then we can run the pipeline via

```shell
./pipeline_demo.sh
```

The final and intermediate data artifacts are in `/tmp/ssa_test` and the trained ML models are in `/tmp/ssa_test/err_models`.

If you would like to see what the resulting conjunction predictions look like, run

```shell
cp /tmp/ssa_test/orbit_preds.pickle ../conjunction_search/sample_data/orbit_preds.pickle
```

then run the development version of the conjunction search UI as detailed [here](../conjunction_search/README.md).
