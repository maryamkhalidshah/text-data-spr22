---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.13.6
  kernelspec:
    display_name: Python [conda env:text-data-class]
    language: python
    name: conda-env-text-data-class-py
---

```python
# Libraries
from dvc.api import read, get_url
import pandas as pd
import numpy as np

import re

import seaborn as sns
import matplotlib.pyplot as plt

import warnings
warnings.filterwarnings('ignore')

from sklearn.tree import DecisionTreeClassifier 
from sklearn.model_selection import train_test_split 
from sklearn import metrics

from sklearn.feature_extraction.text import TfidfVectorizer

```

# Get the Data

```python
# Read from repo using DVC
txt = read('resources/data/shakespeare/shakespeare.txt', 
           repo='https://github.com/TLP-COI/text-data-course')

# # Can also read from local file as wel
# f = open('/Users/chau/text-data-course/shakespeare.txt','r')
# txt = f.read()

print(txt[:250])
```

# Part 1

Split the text file into a table, such that

* each row is a single line of dialogue
* there are columns for

1. the speaker

2. the line number

3. the line dialogue (the text)

Hint: you will need to use RegEx to do this rapidly. See the in-class "markdown" example!

#### Question(s): What assumptions have you made about the text that allowed you to do this?


```python
# Print first 250 letters
txt[:250]
```

### Step 1: Split into individual dialogues

**Assumption 1a:** A dialogue looks like a paragraph - meaning it ends with a double line break <code>\n\n</code>. 

I can use <code>re.split</code> to split the text into dialogues.

```python
# Print first dialogue
re.split(r"\:\n", txt)[0]
```

```python
#Print last dialogue
re.split(r"\n\n", txt)[-1]
```

I will use the text of the last dialogue as a sample for the functions I'll define

```python
# Use last dialogue as sample
sample = re.split(r"\n\n", txt)[-1]

# Pretty print
print(sample)
```

```python
# Ugly print
sample
```

### Step 2: Split dialogue into speaker and dialogue text

```python
# Count number of dialogues in the text
num_dia = len(re.split(r"\n\n", txt))

num_dia
```

**Assumption 1b:** Within a dialogue, speaker name is something that comes before <code>:\n</code> in each split dialogue. I can use <code>^[^:]*</code> to extract it

```python
# Function to get speaker name
def get_speaker_name(dialogue):
    '''Get the speaker name from each dialogue
    Speaker name is defined as anything between the first instance of colon'''
    speaker_name = re.compile(r'^[^:]*').match(dialogue).group()
    return speaker_name
```

```python
# Test the function out
get_speaker_name(sample)
```

**Assumption 1c:** The dialogue content is anything that's not the speaker name in a dialogue. I can try to use a negative look ahead to solve this with regex.

```python
# Function to get dialogue content:
def get_dialogue_content(dialogue):
    '''Get content of the dailogue'''
    dialogue_content = re.split(":", dialogue)[1]
    return dialogue_content
```

**Hacky solution** Since I can't figure out a negative look ahead to get the dialogue content and want to move on with the rest of the homework, I will use <code>re.split</code> to get the speaker name & dialogue. If I have more time next week I will revisit <code>get_dialogue_content</code>


### Step 3: Put it all together

```python
# Function to get the dialogue from the text
def get_dialogue(text):
    '''Input: the full text 
    Returns: a list of list'''
    # Create enumerate object to count line number
    enum = enumerate(re.split(r"\n\n", text))
    
    # List comprehension to put the line number and the 2 custom functions above together 
    res = [[i,get_speaker_name(j),get_dialogue_content(j)] for i,j in enum]
    return res
```

```python
# Check if it works
dia_list = get_dialogue(txt)
dia_list[:3]
```

```python
# Convert to pd
df = pd.DataFrame(dia_list, columns =['line_no','speaker','dialogue'])
# Check 
df[:3]
```

```python
# Remove linebreaks
df = df.replace(r'\n',' ', regex=True) 
```

```python
# Remove trailing/ leading white space from speaker and dialogue
df.speaker = df.speaker.str.strip()
df.dialogue = df.dialogue.str.strip()
```

```python
df[:5]
```

# Part 2

You have likely noticed that the lines are not all from the same play! Now, we will add some useful metadata to our table:

* Determine a likely source title for each line

* add the title as a 'play' column in the data table.

* make sure to document your decisions, assumptions, external data sources, etc.

This is fairly open-ended, and you are not being judged completely on accuracy. Instead, think outside the box a bit as to how you might accomplish this, and attempt to justify whatever approximations or assumptions you felt were appropriate.

```python
# Number of unique speakers
df.speaker.nunique()
```

```python
# List of unique speakers
speaker_list = df.speaker.unique()
```

```python
speaker_list
```

**Assumption 2a** It looks like names written in all caps might be main characters?

```python
# Create a column in df to determine if the speaker name is all caps
df['name_cap']=df.apply(lambda x: x['speaker'].isupper(), axis=1)
```

```python
# 20 speakers with the most lines
df.speaker.value_counts().head(20)
```

```python
# 20 spekers with the fewest line
df.speaker.value_counts().tail(20)
```

```python
# Add number of line count for each speaker in df
df['line_count'] = df.groupby(['speaker'])['line_no'].transform('count')
```

```python
df.head()
```

```python
# Distribution of line count
sns.histplot(df,kde=True, x='line_count')
```

Mahority of characters have very few lines. Next I want to see the distribution split by name capitalization

```python
# Distribution of line count 
sns.histplot(df,kde=True, x='line_count', hue = 'name_cap')
```

It looks like there are characters with a significant number of lines whose name where not capitalized. Check to see who they are.

```python
# Speakers with more than 50 lines & non-cap names:
df.loc[(df.line_count>=50)&(df.name_cap==False)].speaker.unique()
```

Capitalized names are for named characters (not necessarily main?)

```python
# Print the non-cap names
df.loc[(df.name_cap==False)].speaker.unique()
```

**Observation:** There are a few "ghost of" named charactes that are not all caps


For each speaker, I want to get the first line index and last line index - this should give me a general idea of which speaker belongs in which play

```python
# Get first and last line number
df['first_line']=df.groupby(['speaker'],sort=False)['line_no'].transform(min)
df['last_line']=df.groupby(['speaker'],sort=False)['line_no'].transform(max)
```

```python
# Calculate the gap between a character's line and the next 
df['line_gap'] = df.groupby(['speaker'])['line_no'].transform(lambda x: x.diff(periods=-1)).fillna(0)
# Give each character a unique ID number
df['id'] = df.groupby('speaker',sort=False).ngroup()
```

**Assumption 2b** Generic characters without capped names are not unique to each stories, so their first & last line max are not important in determining which plays they belong to. Named-characters whose line_max and line_min overlap are from the same play. I can visualize this.

```python
# Subset out characters with non-cap names & characters with only 1 lines
df_main = df[(df.line_count > 1) & (df.name_cap==True)]
```

```python
# Box plot
plt.figure(figsize = (20,40))
sns.boxplot(data=df_main, x='line_no',y='speaker',dodge=False)
```

From the figure above, I can already pick out some characters that from the same play - the last 9 (from Alonso to Francisco) are probably in the same play because their line numbers overlap and they don't have overlaps with other characters. 

```python
# Check Queen Elizabeth
df_main[df_main.speaker=='QUEEN ELIZABETH'].tail()
```

**Caveat:** Some characters are harder to distinguish - QUEEN ELIZABETH's first line is 1229 and last line is 4398, while only having 105 lines total. According to this [Quora post](https://www.quora.com/Which-character-appears-the-most-in-Shakespeares-plays), Queen Elizabeth appeared in 4 plays and there are other recurring characters in the "Shakespeare verse" as well.

**Solution:** Combine a lollipop plot with a swarm plot to show each character's first and last line, as well as each line they actually spoke.

```python
# roder the data by line index
df_main_ordered = df_main.sort_values(by='line_no')

# The lollipop is made using the hline function
plt.figure(figsize = (20,40))
plt.hlines(y=df_main_ordered['speaker'], xmin=df_main_ordered['first_line'], xmax=df_main_ordered['last_line'], color='lightgray', alpha=0.4)
plt.scatter(df_main_ordered['first_line'], y=df_main_ordered['speaker'], color='black', alpha=1, label='first',marker="|")
plt.scatter(df_main_ordered['last_line'], y=df_main_ordered['speaker'], color='black', alpha=1 , label='last',marker="|")

sns.swarmplot(data=df_main_ordered, x='line_no',y='speaker',size=3,alpha=0.6,dodge=True)

```

The updated chart gives me an idea how to group characters into plays based on line index. Because the visualization is for unique characters, not speaking turn, I can group characters into plays, then try to determine a rule to split line numbers into separate plays. I will do the obvious groups first.

```python
# Similar plot as above, but instead of speaker name I'm using unique speaker ID
plt.figure(figsize = (10,10))
sns.scatterplot(data=df_main,x='line_no',y='id',legend=False)
plt.gca().invert_yaxis()
```

### Clustering?

From the above visualization, I suspect there are 9 plays in the text (at this point I haven't incorporated any outside information aside from that Quora post confirming that there are "recurring" characters in Shakespeareverse). I have tried a few different clustering methods to group the characters and perform visual checks to see if the labels align with what I think the play should be. 

I am trying to clusters of lines based on the line number, the character's unique id number, the first line, last line, total line number and the gap between the current line and the chracter's next line. Since I know that sequential lines are likely from the same play. However, because of the recurring characters, I will set the n_clusters to be higher than 9 so I can capture when a group of recurring characters speak.

This exercise is to see if an algorithm can "color" the points the same way I would color them so it's not an exact science - I'm merely using it as a visual guide. I added vertical lines to see how the labels would split the lines based on number

```python
from sklearn.cluster import KMeans
from sklearn.cluster import AgglomerativeClustering
from sklearn_extra.cluster import KMedoids
from sklearn import cluster 


X= df_main[['id','last_line','first_line','line_count','line_no']]

# Kmeans
# clustering = KMeans(n_clusters=12, init='k-means++', random_state=42).fit(X)
# clustering = KMedoids(n_clusters=12, random_state=42).fit(X)

# Agglomerative
clustering = AgglomerativeClustering(n_clusters=12,linkage='ward').fit(X)

# Create the plot
plt.figure(figsize = (10,10))

plt.gca().invert_yaxis()

sns.scatterplot(data=df_main,x='line_no',y='id',color=None,edgecolor=None,hue=clustering.labels_,palette='deep',alpha=1)

# Attach cluster labels to df_main
df_main['cluster'] = clustering.labels_

# Get the min line number for each cluster
cluster_min =[]
for i in range(12):
    cmin = df_main[df_main.cluster==i]['line_no'].min()
    cluster_min.append(cmin)

# Plot vertical lines to do visual check
for xc in cluster_min:
    plt.axvline(x=xc, color='black', linewidth=0.5)
```

```python
sorted(cluster_min)
```

I'm trying to estimate the play splits based on the cluster_min. From the sorted list above and the figure, I decided to split the dataset into plays at the following line numbers

```python
# split at points
plays = [1107, 2103, 2876, 3503,4403,5149, 6120, 6943, 7222]
```

```python
# Function to get play number:
def play_no(row):
    x= 0
    for i,v in enumerate(plays):
        if row in range(x,v):
            label = i
            x = v
            return label
```

```python
# See if it works
play_no(7221)
```

```python
# Apply it to df and df_main
df['play_no'] = df['line_no'].apply(lambda x: play_no(x))
df_main['play_no'] = df_main['line_no'].apply(lambda x: play_no(x))
```

Now, I want to see how my clustered labels perform witht the full text which includes non-main/ non-unique characters

```python
# Re-create the plot for the full df (which includes non-main characters)
plt.figure(figsize = (10,10))

plt.gca().invert_yaxis()
sns.scatterplot(data=df,x='line_no',y='id',color=None,edgecolor=None,hue='play_no',palette='deep',alpha=1)

```

This does not look bad - and I have not used any actual textual data or outside source yet. Because I did not know the outcome variable (names of plpays), I chose to cluster the characters based on line numbers and other lin-number-related features to see if I could use clustering analysis. 


# Check external sources

However, since Shakespeare plays are not unknown, I now look for outside sources to examine other ways to classify characters and lines. 

I downloaded the csv for all of Shakespeare's lines from [Kaggle](https://www.kaggle.com/kingburrito666/shakespeare-plays?select=Shakespeare_data.csv)

```python
# Read in this data
sp = pd.read_csv("./kaggle/Shakespeare_data.csv")

sp.head()
```

```python
len(sp)
```

The kaggle dataset is quite similar to the tiny_shakespeare data I had before, but it has more details: 

1) there are Scence and Act numbers

2) Each line in a character's dialogue is broken into a separate row, whereas I broke each dialogue in the tiny_shakespeare dataset into a "document"

**Task:** I want to transform the sp dataset so that it somewhat resembles my df.

```python
# group lines into the same dialogue to make a "document"
sp['dialogue'] = sp.groupby(['Play','Player','PlayerLinenumber'])['PlayerLine'].transform(lambda x : ' '.join(x))
sp.head()
```

```python
# drop dulicate lines
sp = sp.drop_duplicates(subset=['Play','PlayerLinenumber','Player'])

```

```python
# Drop PlayerLine, index and Dataline
sp = sp.drop(columns = ['Dataline','PlayerLine'])
```

```python
sp.head()
```

```python
# Drop lines where Player is NaN
sp = sp[sp.Player.notna()]

# Reset index
sp = sp.reset_index(drop=True)

sp.head()
```

```python
# Check number of rows after combining lines & dropping lines I don't need
len(sp)
```

```python
# How many plays are in sp
sp.Play.nunique()
```

```python
# How many unique Players
sp.Player.nunique()
```

```python
sp['player'] = sp.Player.str.lower()
```

```python
sp.player.nunique()
```

sp has way more players/speakers than df


Because tiny_shakespeare does not include all of Shakespeare's works, I have a few options:

1) Drop rows from sp where sp.Player is not in df.speaker, then do a one to one match to determine exactly which line belongs to which play

2) Cluster the lines in sp, then use it to predict lines in df

Eventually, I also want to compare it to the manual labels I gave lines in df earlier


### Shakespeare classifier?

I decided to use the large sp data set to train a classifier to determine the name of Shakespeare's plays. 

```python
# Convert Play and Player in sp into numeric values
sp['Player_id'] = sp.groupby('Player',sort=False).ngroup()
sp['Play_id'] = sp.groupby('Play',sort=False).ngroup()
```

```python
# # Note: loosely based on this: https://github.com/tmhughes81/dap/blob/master/dap.ipynb

# # Vectorize dialogues from the entire Shakespeare corpus
# vec = TfidfVectorizer()
# tf_idf_vec = vec.fit_transform(sp['dialogue'])

# # Convert the tf_idf matrix into a data frame with labels
# tf_idf_df = pd.DataFrame(columns=vec.get_feature_names(), data=tf_idf_vec.toarray())

# # Convert the tf_idf matrix into a data frame with labels
# tf_idf_df = pd.DataFrame(columns=vec.get_feature_names(), data=tf_idf_vec.toarray())
```

```python
# # drop dialogue from sp
# sp_joined = sp[['Player_id','Play_id','Player','Play']]

# # Combine the original data set with the tf_idf dataframe
# sp_joined = sp_joined.join(tf_idf_df)
```

```python
# sp_joined
```

#### Create training, validation and test data

```python
# Create empty data frames
train = pd.DataFrame({'Play_id':[], 'Player_id':[], 'dialogue':[]})
test = pd.DataFrame({'Play_id':[], 'Player_id':[], 'dialogue':[]})
```

I don't want to do a pure train/test split on the entire sp data because I want to make sure that the training/ validation/ test sets have samples from every play

```python
# Split sp into train and test df
for i in sp.Play_id.unique():
    # Get rows of each individual play
    temp = sp[sp.Play_id==i]
    # Do a train test split for each play so the training & test set has rows from all 
    temp_train, temp_test = train_test_split(temp, test_size = 0.2, random_state = 42)
    
    # Append to the empty dataframe
    train = train.append(temp_train)
    test =test.append(temp_test)

```

```python
# Split features
# X_train = train.drop(columns=['Play_id'])
# X_test = test.drop(columns=['Play_id'])

X_train = train[['Player_id']]
X_test = test[['Player_id']]

y_train = train['Play_id']
y_test = test['Play_id']
```

```python
dt = DecisionTreeClassifier()
dt.fit(X_train, y_train)
y_pred = dt.predict(X_test)
```

```python
print("Accuracy:",metrics.accuracy_score(y_test, y_pred))
```

### Back to df

```python
player_df = sp.drop_duplicates(subset=['Player','Player_id'])
player_dict = dict(zip(player_df.Player, player_df.Player_id))
```

```python
player_dict
```

```python
# Subset df into something cleaner but keeping play_no from clustering above for prosperity 
tiny_df = df[['line_no','speaker','dialogue','play_no']]

```

```python
# Map Player_id from sp to df
tiny_df['Player_id'] = tiny_df.speaker.map(player_dict)
tiny_df.head()
```

```python
# Check for NAs
tiny_df[tiny_df.Player_id.isna()]
```

The NAs are from so few lines that I think I'll drop them *for now* so I can move on with the rest of the homework

```python
tiny_df = tiny_df[tiny_df.Player_id.notna()]
# Re-map
tiny_df['Player_id'] = tiny_df.speaker.map(player_dict)

tiny_df.tail()
```

```python
#Predict
tiny_pred = dt.predict(tiny_df[['Player_id']])
```

```python
tiny_df.Player_id.nunique()
```

### Part 3

Pick one or more of the techniques described in this chapter:

* keyword frequency

* entity relationships
    
* markov language model
    
* bag-of-words, TF-IDF
    
* semantic embedding

make a case for a technique to measure how important or interesting a speaker is. The measure does not have to be both important and interesting, and you are welcome to come up with another term that represents "useful content", or tells a story (happiest speaker, worst speaker, etc.)

Whatever you choose, you must

1. document how your technique was applied

2. describe why you believe the technique is a valid approximation or exploration of how important, interesting, etc., a speaker is.

3. list some possible weaknesses of your method, or ways you expect your assumptions could be violated within the text.

This is mostly about learning to transparently document your decisions, and iterate on a method for operationalizing useful analyses on text. Your explanations should be understandable; homeworks will be peer-reviewed by your fellow students.




```python

```

```python

```

```python

```

```python

```

```python

```

```python

```

```python

```

```python

```

```python

```

```python

```

```python
df
```

```python

```

```python

```

```python

```

```python

```

```python

```
