---
title: Globalize the scope of a python function
summary: "Globalize the scope of a python function"
date: "2023-09-24"
tags:
    - Python
---

In the last month, I faced an interesting problems using Python that shed light on an interesting aspect of this language.

## The use case

At Sunrise I worked with sleep recordings to detect apneas and respiratory efforts using machine learning models. Some of our data pipelines exploit the predictions to generate some useful clinical socres that permits to evaluate the quality of the night. Regularly and to follow up and the quality of our models, we caclulate the scores from a subsequent portion of our historical nights recorded with our medical device and their respective labeled data from classical pletysmography. The pipeline is deployed on Vertex.ai and is articulated in several steps:

- Gather all labeled data (hypnograms, respiratory events, head motion)
- Do some processing of the labeled data (column naming, mapping of the labels) and concatenate all nights
- Calculate scores from labeled data and our predictions
- Generate a full report comparing and evaluating the scores from our predictions and lebeled data.

The first step was quite long and intensive. The algorithm for processing was loading one by one each `csv` file containing the labeled data and concatenating all into one big dataframe. We had 2200 nights with files with almost 100000 lines each. It was roughly taking 30 to 60 minutes to process our different files.

## The code

The processing step was implemented as a [Vertex.ai](https://fbraza.github.io/fbraza-github-pages/overview-of-vertexai/) pipeline's step. It was very similar to the following:

```python
@component(
    packages_to_install=["any", "library", "you"],
    base_image="uri/to/image",
)
def concatenate_hypnograms(
    labels_path: str, concatenated_hypnos: Output[Dataset]
):
    from datetime import datetime

    import pandas
    import os

    nights_hypnograms = os.listdir("path/to/hypnograms/csv")
    dataframes = []
    for night in nights_hypnograms:
      df = pd.read_csv(f"parent/{night}")
      # Do other things
      dataframes.append(df)

    final_df = pd.concatenate(dataframes, axis=0)
```

It is easy to understand that the code is running at O(n) and that time complexity will increase with the number of nights. To speed up the process I investigated the use of multiprocessing and start implementing the code:

```python
from multiprocessing import Pool


def parralel_read_csv_for_labels(labels_path: str) -> pd.DataFrame:
    file_list = {
        name
        for name in glob.glob(f"{labels_path}/**", recursive=True)
    }

    with Pool() as pool:
        df_list = pool.starmap(
            add_night_column_for_labels,
            zip(file_list, itertools.repeat(sunalgo_labels_path)),
        )
        combined_df = pd.concat(df_list, axis=0)

    return combined_df
```

A code very similar to this one, was implemented in our in-house library. We could then just import it in our component:

```python
@component(
    packages_to_install=["any", "library", "you"],
    base_image="uri/to/image",
)
def concatenate_hypnograms(
    labels_path: str, concatenated_hypnos: Output[Dataset]
):
    from datetime import datetime

    import pandas
    import os
    from library import parralel_read_csv_for_labels

    final_df = parralel_read_csv_for_labels(labels_path=labels_path)
```

Executing our code returned the following error:

```bash
_pickle.PicklingError: Can't pickle <class '_thread.lock'>: attribute lookup lock on _thread failed
```

This error is due to the fact that importing the function inside my component changed its scope. Indeed, `multiprocessing` uses pickle to package a function in order to send it to other workers, but it doesn't work for non-top-level functions. The trick was to make beleive the workers that the functions was actually at the top level of the scope. To do that I used the following decorator:

```python
import sys
import uuid
from functools import wraps


def globalize(func):
    """
    Function defined to reset the scope hierachy and make an inner function
    considered as a top-level function. We need this in order to use multiprocess
    Pool to parallelize tasks with inner functions in our kubeflow components.
    """

    @wraps(func)
    def result(*args, **kwargs):
        return func(*args, **kwargs)

    result.__name__ = result.__qualname__ = uuid.uuid4().hex
    setattr(sys.modules[result.__module__], result.__name__, result)
    return result
```

`globalize` effectively clones the function, gives the clone a unique name, and inserts the clone as a top-level function into the original function's module. The nice thing is that the clone retains the original function's context, allowing it to access variables that the original function can. By refactoring the previous code we could speed up our pipeline steps going from 30-60 minutes to 5-10 minutes processing time.

```python
@component(
    packages_to_install=["any", "library", "you"],
    base_image="uri/to/image",
)
def concatenate_hypnograms(
    labels_path: str, concatenated_hypnos: Output[Dataset]
):
    from datetime import datetime

    import pandas
    import os
    from library import parralel_read_csv_for_labels, globalize

    parralel_read_csv_for_labels = globalize(parralel_read_csv_for_labels)
    final_df = parralel_read_csv_for_labels(labels_path=labels_path)
```

## Final thoughts

The primary objective was to speed up this processing step. Using this approach worked effciently. There is another approach that was not investigated and could be more elegant: the `[ParallelFor](https://kubeflow-pipelines.readthedocs.io/en/sdk-2.2.0/source/dsl.html?h=parallelfor#kfp.dsl.ParallelFor)` class provided by the kubeflow sdk. This would imply refactoring the code to exclude any concatenation and treat all files as an indepednant set of data and aggregate everything at the scores computations and reporting steps.

## References
- https://kubeflow-pipelines.readthedocs.io/en/sdk-2.2.0/index.html
- https://docs.python.org/3/library/multiprocessing.html
- https://realpython.com/primer-on-python-decorators/
