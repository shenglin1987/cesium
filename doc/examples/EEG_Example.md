# Epilepsy Detection Using EEG Data

In this example we'll use the [`mltsp`](http://github.com/mltsp/mltsp/) library to compare
various techniques for epilepsy detection using a classic EEG time series dataset from
[Andrzejak et al.](http://www.meb.uni-bonn.de/epileptologie/science/physik/eegdata.html).
The raw data are separated into five classes: Z, O, N, F, and S; we will consider a
three-class classification problem of distinguishing normal (Z, O), interictal (N, F), and
ictal (S) signals.

The overall workflow consists of three steps: first, we "featurize" the time series by
selecting some set of mathematical functions to apply to each; next, we build some
classification models which use these features to distinguish between classes;
finally, we validate our models by generating predictions for some unseen
holdout set and comparing them to the true class labels.

First, we'll load the data and inspect a representative time series from each class:

```python
%matplotlib inline
import numpy as np
import matplotlib.pyplot as plt
import seaborn; seaborn.set()

from mltsp import datasets

eeg = datasets.fetch_andrzejak()

# Group together classes (Z, O), (N, F), (S) as normal, interictal, ictal
eeg["classes"] = eeg["classes"].astype('U16') #  allocate memory for longer class names
eeg["classes"][np.logical_or(eeg["classes"]=="Z", eeg["classes"]=="O")] = "Normal"
eeg["classes"][np.logical_or(eeg["classes"]=="N", eeg["classes"]=="F")] = "Interictal"
eeg["classes"][eeg["classes"]=="S"] = "Ictal"

fig, ax = plt.subplots(1, len(np.unique(eeg["classes"])), sharey=True)
for label, subplot in zip(np.unique(eeg["classes"]), ax):
    i = np.where(eeg["classes"] == label)[0][0]
    subplot.plot(eeg["times"][i], eeg["measurements"][i])
    subplot.set(xlabel="time (s)", ylabel="signal", title=label)
```

## Featurization
Once the data is loaded, we can generate features for each time series using the
[`mltsp.featurize`](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html)
module. The `featurize` module includes many built-in choices of features which can be applied
for any type of time series data; here we've chosen a few generic features that do not have
any special biological significance.

If Celery is running, the time series will automatically be split among the available workers
and featurized in parallel; setting `use_celery=False` will cause the time series to be
featurized serially.

```python
from mltsp import featurize
features_to_use = ['amplitude',
                   'percent_beyond_1_std',
                   'maximum',
                   'max_slope',
                   'median',
                   'median_absolute_deviation',
                   'percent_close_to_median',
                   'minimum',
                   'skew',
                   'std',
                   'weighted_average']
fset_mltsp = featurize.featurize_time_series(times=eeg["times"], values=eeg["measurements"],
                                             errors=None, features_to_use=features_to_use,
                                             targets=eeg["classes"], use_celery=True)
print(fset_mltsp)
```

The output of
[`featurize_time_series`](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html#mltsp.featurize.featurize_time_series)
is an `xarray.Dataset` which contains all the feature information needed to train a machine
learning model: feature values are stored as data variables, and the time series index/class
label are stored as coordinates (a `channel` coordinate will also be used later for
multi-channel data).

### Custom feature functions
Custom feature functions not built into `mltsp` may be passed in using the
`custom_functions` keyword, either as a dictionary `{feature_name: function}`, or as a
[dask graph](http://dask.pydata.org/en/latest/custom-graphs.html). Functions should take
three arrays `times, measurements, errors` as inputs; details can be found in the
`mltsp.featurize`
[documentation](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html).
Here we'll compute five standard features for EEG analysis provided by [Guo et al.
(2012)](http://linkinghub.elsevier.com/retrieve/pii/S0957417411003253):

```python
import numpy as np
import scipy.stats

def mean_signal(t, m, e):
    return np.mean(m)

def std_signal(t, m, e):
    return np.std(m)

def mean_square_signal(t, m, e):
    return np.mean(m ** 2)

def abs_diffs_signal(t, m, e):
    return np.sum(np.abs(np.diff(m)))

def skew_signal(t, m, e):
    return scipy.stats.skew(m)
```

Now we'll pass the desired feature functions as a dictionary via the `custom_functions`
keyword argument.

```python
guo_features = {
    'mean': mean_signal,
    'std': std_signal,
    'mean2': mean_square_signal,
    'abs_diffs': abs_diffs_signal,
    'skew': skew_signal
}

fset_guo = featurize.featurize_time_series(times=eeg["times"], values=eeg["measurements"],
                                           errors=None, targets=eeg["classes"], 
                                           features_to_use=list(guo_features.keys()),
                                           custom_functions=guo_features,
                                           use_celery=True)
print(fset_guo)
```

### Multi-channel time series
The EEG time series considered here consist of univariate signal measurements along a
uniform time grid. But
[`featurize_time_series`](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html#mltsp.featurize.featurize_time_series)
also accepts multi-channel data; to demonstrate this, we will decompose each signal into
five frequency bands using a discrete wavelet transform as suggested by [Subasi
(2005)](http://www.sciencedirect.com/science/article/pii/S0957417404001745), and then
featurize each band separately using the five functions from above.

```python
import pywt

n_channels = 5
eeg["dwts"] = [pywt.wavedec(m, pywt.Wavelet('db1'), level=n_channels-1)
               for m in eeg["measurements"]]
fset_dwt = featurize.featurize_time_series(times=None, values=eeg["dwts"], errors=None,
                                           features_to_use=list(guo_features.keys()),
                                           targets=eeg["classes"],
                                           custom_functions=guo_features)
print(fset_dwt)
```

The output featureset has the same form as before, except now the `channel` coordinate is
used to index the features by the corresponding frequency band. The functions in
[`mltsp.build_model`](https://mltsp.readthedocs.org/en/latest/api/mltsp.build_model.html)
and [`mltsp.predict`](https://mltsp.readthedocs.org/en/latest/api/mltsp.predict.html)
all accept featuresets from single- or multi-channel data, so no additional steps are
required to train models or make predictions for multichannel featuresets using the
`mltsp` library.

## Model Building
Model building in `mltsp` is handled by the
[`build_model_from_featureset`](https://mltsp.readthedocs.org/en/latest/api/mltsp.build_model.html#mltsp.build_model.build_model_from_featureset)
function in the `mltsp.build_model` submodule. The featureset output by
[`featurize_time_series`](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html#mltsp.featurize.featurize_time_series)
contains both the feature and target information needed to train a
model; `build_model_from_featureset` is simply a wrapper that calls the `fit` method of a
given `scikit-learn` model with the appropriate inputs. In the case of multichannel
features, it also handles reshaping the featureset into a (rectangular) form that is
compatible with `scikit-learn`.

For this example, we'll test a random forest classifier for the built-in `mltsp` features,
and a 3-nearest neighbors classifier for the others, as suggested by [Guo et al.
(2012)](http://linkinghub.elsevier.com/retrieve/pii/S0957417411003253).

```python
from mltsp.build_model import build_model_from_featureset
from sklearn.ensemble import RandomForestClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.cross_validation import train_test_split

train, test = train_test_split(np.arange(len(eeg["classes"])), random_state=0)

rfc_param_grid = {'n_estimators': [8, 16, 32, 64, 128, 256, 512, 1024]}
model_mltsp = build_model_from_featureset(fset_mltsp.isel(name=train),
                                          RandomForestClassifier(max_features='auto',
                                                                 random_state=0),
                                          params_to_optimize=rfc_param_grid)
knn_param_grid = {'n_neighbors': [1, 2, 3, 4]}
model_guo = build_model_from_featureset(fset_guo.isel(name=train),
                                        KNeighborsClassifier(),
                                        params_to_optimize=knn_param_grid)
model_dwt = build_model_from_featureset(fset_dwt.isel(name=train),
                                        KNeighborsClassifier(),
                                        params_to_optimize=knn_param_grid)
```

## Prediction
Making predictions for new time series based on these models follows the same pattern:
first the time series are featurized using
[`featurize_timeseries`](https://mltsp.readthedocs.org/en/latest/api/mltsp.featurize.html#mltsp.featurize.featurize_time_series),
and then predictions are made based on these features using
[`predict.model_predictions`](https://mltsp.readthedocs.org/en/latest/api/mltsp.predict.html#mltsp.predict.model_predictions),

```python
from sklearn.metrics import accuracy_score
from mltsp.predict import model_predictions

preds_mltsp = model_predictions(fset_mltsp, model_mltsp, return_probs=False)
preds_guo = model_predictions(fset_guo, model_guo, return_probs=False)
preds_dwt = model_predictions(fset_dwt, model_dwt, return_probs=False)

print("Built-in MLTSP features: training accuracy={:.2%}, test accuracy={:.2%}".format(
          accuracy_score(preds_mltsp[train], eeg["classes"][train]),
          accuracy_score(preds_mltsp[test], eeg["classes"][test])))
print("Guo et al. features: training accuracy={:.2%}, test accuracy={:.2%}".format(
          accuracy_score(preds_guo[train], eeg["classes"][train]),
          accuracy_score(preds_guo[test], eeg["classes"][test])))
print("Wavelet transform features: training accuracy={:.2%}, test accuracy={:.2%}".format(
          accuracy_score(preds_dwt[train], eeg["classes"][train]),
          accuracy_score(preds_dwt[test], eeg["classes"][test])))
```

The workflow presented here is intentionally simplistic and omits many important steps
such as feature selection, model parameter selection, etc. Since we make use of standard
`scikit-learn` models, additional steps can be incorporated in the same way they would be
for any other (non-time domain) machine learning problem.


IPython notebook: <a href="EEG_Example.ipynb" download="EEG_Example.ipynb">download</a>