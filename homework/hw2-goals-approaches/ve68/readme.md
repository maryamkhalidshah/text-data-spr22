---
jupytext:
  formats: ipynb,md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.13.6
kernelspec:
  display_name: Python [conda env:text-data-class]
  language: python
  name: conda-env-text-data-class-py
---

# Homework: Goals & Approaches

> The body grows stronger under stress. The mind does not.
> 
>  -- Magic the Gathering, _Fractured Sanity_

This homework deals with the goals you must define, along with the approaches you deem necessary to achieve those goals. 
Key to this will be a focus on your _workflows_: 

- are they reproducible? 
- are they maintainable? 
- are they well-justified and communicated? 

This is not a "machine-learning" course, but machine learning plays a large part in modern text analysis and NLP. 
Machine learning, in-turn, has a number of issues tracking and solving issues in a collaborative, asynchronous, distributed manner. 

It's not inherently _wrong_ to use pre-configured models and libraries! 
In fact, you will likely be unable to create a set of ML algorithms that "beat" something others have spent 100's of hours creating, optimizing, and validating. 
However, to answer the three questions above, we need a way to explicitly track our decisions to use others' work, and efficiently _swap out_ that work for new ideas and directions as the need arises. 

This homework is a "part 1" of sorts, where you will construct several inter-related pipelines in a way that will allow _much easier_ adjustment, experimentation, and measurement in "part 2"

+++

## Setup

### Dependencies 
As before, ensure you have an up-to-date environment to isolate your work. 
Use the `environment.yml` file in the project root to create/update the `text-data-class` environment. 
> I expect any additional dependencies to be added here, which will show up on your pull-request. 

### Data
Once again, we have set things up to use DVC to import our data. 
If the data changes, things will automatically update! 
The data for this homework has been imported as `mtg.feather` under the `data/` directory at the top-level of this repository. 
In order to ensure your local copy of the repo has the actual data (instead of just the `mtg.feather.dvc` stub-file), you need to run `dvc pull`

```{code-cell} ipython3
!dvc pull
```

Then you may load the data into your notebooks and scripts e.g. using pandas+pyarrow:

+++

But that's not all --- at the end of this homework, we will be able to run a `dvc repro` command and all of our main models and results will be made available for your _notebook_ to open and display.

+++

### Submission Structure
You will need to submit a pull-request on DagsHub with the following additions: 

- your subfolder, e.g. named with your user id, inside the `homework/hw2-goals-approaches/` folder
    - your "lab notebook", as an **`.ipynb` or `.md`** (e.g. jupytext), that will be exported to PDF for Canvas submission. **This communicates your _goals_**, along with the results that will be compared to them. 
    - your **`dvc.yaml`** file that will define  the inputs and outputs of your _approaches_. See [the DVC documentation](https://dvc.org/doc/user-guide/project-structure/pipelines-files) for information!
    - **source code** and **scripts** that define the preprocessing and prediction `Pipeline`'s you wish to create. You may then _print_ the content of those scripts at the end of your notebook e.g. as appendices using 
- any updates to `environment.yml` to add the dependencies you want to use for this homework

```{code-cell} ipython3
import pandas as pd

df = pd.read_feather("../../../data/mtg.feather")
```

## Part 1: Unsupervised Exploration

Investigate the [BERTopic](https://maartengr.github.io/BERTopic/index.html) documentation (linked), and train a model using their library to create a topic model of the `flavor_text` data in the dataset above. 

- In a `topic_model.py`, load the data and train a bertopic model. You will `save` the model in that script as a new trained model object
- add a "topic-model" stage to your `dvc.yaml` that has `mtg.feather` and `topic_model.py` as dependencies, and your trained model as an output
- load the trained bertopic model into your notebook and display
    1. the `topic_visualization` interactive plot [see docs](https://maartengr.github.io/BERTopic/api/plotting/topics.html)
    2. Use the plot to come up with working "names" for each major topic, adjusting the _number_ of topics as necessary to make things more useful. 
    3. Once you have names, create a _Dynamic Topic Model_ by following [their documentation](https://maartengr.github.io/BERTopic/getting_started/topicsovertime/topicsovertime.html). Use the `release_date` column as timestamps. 
    4. Describe what you see, and any possible issues with the topic models BERTopic has created. **This is the hardest part... interpreting!**

```{code-cell} ipython3
from bertopic import BERTopic

ft_topic_model = BERTopic.load("flavor_text_topics")
```

```{code-cell} ipython3
ft_topic_model.visualize_topics(
    top_n_topics=11)
```

### Names for Top 10 Topics

1. Sword
2. Goblins
3. [Planet] Ravnica
4. [Plane] Kamigawa
5. Mages
6. Hunter
7. Necromancy
8. Dragon
9. Rakdos
10. Sarpadian [Empire]

```{code-cell} ipython3
df_ft_rd = df[['flavor_text', 'release_date']].dropna().reset_index(drop=True)
```

```{code-cell} ipython3
topics, probs = ft_topic_model.transform(df_ft_rd['flavor_text'])
```

```{code-cell} ipython3
topics_over_time = ft_topic_model.topics_over_time(df_ft_rd['flavor_text'], topics, df_ft_rd['release_date'])
```

```{code-cell} ipython3
ft_topic_model.visualize_topics_over_time(topics_over_time, top_n_topics=20)
```

## Part 2: Supervised Classification

Using only the `text` and `flavor_text` data, predict the color identity of cards: 

Follow the sklearn documentation covered in class on text data and Pipelines to create a classifier that predicts which of the colors a card is identified as. 
You will need to preprocess the target _`color_identity`_ labels depending on the task: 

- Source code for pipelines
    - in `multiclass.py`, again load data and train a Pipeline that preprocesses the data and trains a multiclass classifier (`LinearSVC`), and saves the model pickel output once trained. target labels with more than one color should be _unlabeled_! 
    - in `multilabel.py`, do the same, but with a multilabel model (e.g. [here](https://scikit-learn.org/stable/auto_examples/miscellaneous/plot_multilabel.html#sphx-glr-auto-examples-miscellaneous-plot-multilabel-py)). You should now use the original `color_identity` data as-is, with special attention to the multi-color cards. 
- in `dvc.yaml`, add these as stages to take the data and scripts as input, with the trained/saved models as output. 

- in your notebook: 
    - Describe:  preprocessing steps (the tokenization done, the ngram_range, etc.), and why. 
    - load both models and plot the _confusion matrix_ for each model ([see here for the multilabel-specific version](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.multilabel_confusion_matrix.html))
    - Describe: what are the models succeeding at? Where are they struggling? How do you propose addressing these weaknesses next time?

```{code-cell} ipython3
# Load scikit-learn model
import pickle

# Processing
from sklearn import preprocessing
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.model_selection import train_test_split

# Evaluation
from sklearn.metrics import confusion_matrix
from sklearn.metrics import multilabel_confusion_matrix

# Plotting
import matplotlib.pyplot as plt
import seaborn as sns
```

```{code-cell} ipython3
multiclass_model = pickle.load(open('multiclass.sav', 'rb'))
```

```{code-cell} ipython3
text = df['text'] + df['flavor_text'].fillna('')

tfidf = TfidfVectorizer(
    min_df=5, 
    stop_words='english')

X_tfidf = tfidf.fit_transform(text)

ci = df['color_identity']
```

```{code-cell} ipython3
single_color_identity = [list(i)[0] if len(i) == 1 else 0 for i in ci]
    
le = preprocessing.LabelEncoder()
y_mc = le.fit_transform(single_color_identity)

X_train, X_test, y_mc_train, y_mc_test = train_test_split(X_tfidf, y_mc, random_state = 2022)
```

```{code-cell} ipython3
y_mc_pred = multiclass_model.predict(X_test)

cf_mc = confusion_matrix(y_mc_test, y_mc_pred)

fig, ax = plt.subplots(figsize=(10,10))
sns.heatmap(cf_mc, annot=True, fmt='d')
plt.ylabel('Actual')
plt.xlabel('Predicted')
plt.show()
```

```{code-cell} ipython3
multilabel_model = pickle.load(open('multilabel.sav', 'rb'))
```

```{code-cell} ipython3
cv = CountVectorizer(tokenizer=lambda x: x, lowercase=False)
y_ml = cv.fit_transform(ci)

X_train, X_test, y_ml_train, y_ml_test = train_test_split(X_tfidf, y_ml, random_state = 20220418)
```

```{code-cell} ipython3
y_ml_pred = multilabel_model.predict(X_test)
```

```{code-cell} ipython3
cf_ml = multilabel_confusion_matrix(y_ml_test, y_ml_pred)
```

#### Reference
https://stackoverflow.com/questions/62722416/plot-confusion-matrix-for-multilabel-classifcation-python

```{code-cell} ipython3
def print_confusion_matrix(confusion_matrix, axes, class_label, class_names, fontsize=14):

    df_cm = pd.DataFrame(
        confusion_matrix, index=class_names, columns=class_names,
    )

    try:
        heatmap = sns.heatmap(df_cm, annot=True, fmt="d", cbar=False, ax=axes)
    except ValueError:
        raise ValueError("Confusion matrix values must be integers.")
        
    heatmap.yaxis.set_ticklabels(heatmap.yaxis.get_ticklabels(), rotation=0, ha='right', fontsize=fontsize)
    heatmap.xaxis.set_ticklabels(heatmap.xaxis.get_ticklabels(), rotation=0, ha='right', fontsize=fontsize)
    axes.set_ylabel('True label')
    axes.set_xlabel('Predicted label')
    axes.set_title("Confusion Matrix for the class - " + class_label)
```

```{code-cell} ipython3
fig, ax = plt.subplots(5, 1, figsize=(8, 8))
    
for axes, cfs_matrix, label in zip(ax.flatten(), cf_ml, cv.get_feature_names_out()):
    print_confusion_matrix(cfs_matrix, axes, label, ["N", "Y"])
    
fig.tight_layout()
plt.show()
```

## Part 3: Regression?

> Can we predict the EDHREC "rank" of the card using the data we have available? 

- Like above, add a script and dvc stage to create and train your model
- in the notebook, aside from your descriptions, plot the `predicted` vs. `actual` rank, with a 45-deg line showing what "perfect prediction" should look like. 
- This is a freeform part, so think about the big picture and keep track of your decisions: 
    - What model did you choose? Why? 
    - What data did you use from the original dataset? How did you preprocess it? 
    - Can we see the importance of those features? e.g. logistic weights? 
    
How did you do? What would you like to try if you had more time?

```{code-cell} ipython3
import numpy as np

from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn import preprocessing
from sklearn.model_selection import train_test_split
    
from sklearn.linear_model import ElasticNet, ElasticNetCV
```

```{code-cell} ipython3
df = pd.read_feather("../../../data/mtg.feather")
```

```{code-cell} ipython3
df
```

```{code-cell} ipython3
df = df[df['edhrec_rank'].notna()]
```

```{code-cell} ipython3
y = df['edhrec_rank']
```

```{code-cell} ipython3
df['text_flavor_text'] = df['text'] + df['flavor_text'].fillna('')

tfidf = TfidfVectorizer(
    min_df=5, 
    stop_words='english')

X_tfidf = tfidf.fit_transform(df['text_flavor_text'])
```

```{code-cell} ipython3
X_train, X_test, y_train, y_test = train_test_split(X_tfidf, y, test_size=0.20, random_state=2022)
```

```{code-cell} ipython3
regression_model = ElasticNetCV(cv=5, random_state=0)
    
regression_model.fit(X_train, y_train)
```

```{code-cell} ipython3
y_pred = regression_model.predict(X_test)
```

```{code-cell} ipython3
sns.scatterplot(x=y_test, y=y_pred)
```

```{code-cell} ipython3

```