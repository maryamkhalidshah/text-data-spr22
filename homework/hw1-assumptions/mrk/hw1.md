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
    display_name: Python 3
    language: python
    name: python3
---

# HW1
Madeline Kinnaird

```python
import pandas as pd
import re
import random

import requests 
from bs4 import BeautifulSoup 

import fuzzywuzzy
```

##### import data

```python
file = open("shakespeare.txt")
txt = file.read()
file.close()
```

# Part 1
Split the text file into a table, such that:
- each row is a single line of dialogue
- there are columns for:
    - the speaker
    - the line number
    - the line dialogue (the text)
    
Hint: you will need to use RegEx to do this rapidly. See the in-class "markdown" example!



```python
n = []
t = [] 
    
list_of_quotes = re.split(r'\n\n',txt) #find each dialog #seperate each dialog
for quote in list_of_quotes:
    #print('**',quote)
    name = re.split(':',quote,1)[0]
    #print('!!!!',name)#[0]#find name
    text = re.split(':',quote,1)[-1].replace("\n", ' ')
    n.append(name)
    t.append(text)
    
shakespeare = pd.DataFrame(n,t).reset_index().rename(columns={"index": "Dialogue", 0: "Speaker"})

## change text to lower
shakespeare = shakespeare.applymap(str.lower)

## add lines numbers
shakespeare = shakespeare.reset_index().rename(columns={"index": "Line Number"})
shakespeare['Line Number'] = shakespeare['Line Number'] + 1
shakespeare = shakespeare[['Speaker', 'Dialogue', 'Line Number']]


shakespeare
```

##### Question(s):
What assumptions have you made about the text that allowed you to do this?

- each quote is separated by two line breaks
- the speaker always ends with :


# Part 2
You have likely noticed that the lines are not all from the same play! Now, we will add some useful metadata to our table:

- Determine a likely source title for each line
- add the title as a 'play' column in the data table.
- make sure to document your decisions, assumptions, external data sources, etc.

This is fairly open-ended, and you are not being judged completely on accuracy. Instead, think outside the box a bit as to how you might accomplish this, and attempt to justify whatever approximations or assumptions you felt were appropriate.


##### External Data Source
A list of all Shakespearan characters and their respective plays.

```python
## set URL
url = "https://www.behindthename.com/namesakes/list/shakespeare/play"
page = requests.get(url)
page.status_code # 200 == Connection
soup = BeautifulSoup(page.content, 'html.parser')

## convert table to dataframe
table = soup.find_all('table')
df = pd.read_html(str(table), header=0)[2]
```

```python
df = df.applymap(str)
df = df.applymap(str.lower)
df.sample(5)
```

##### Make a 'bag of words' type column 
This column is a list of each token in the rest of the row in addition to each phrase in it's entireity. 

```python
def row_function(row):
    ## split the play and name columns into tokens, create list
    l = row['Play'].split(' ') + row['Name'].split(' ')
    ## add the full phrase of name and play 
    l.append(row['Play'])
    l.append(row['Name'])
    ## add non-empty 'other names'/nicknames
    if row['Other Names'] != 'nan':
        l.append(re.compile("\w+").findall(row['Other Names']))
              
    return l
```

```python
df['bag_of_words'] = df.apply(lambda row : row_function(row), axis=1)
df = df.drop(['Other Names', 'Unnamed: 3'], axis=1)
```

```python
df
```

##### Functions to assign most play origination
Checks the speaker name from the shakespeare text and counts the number of times it appears in the bag of words. It then aggregates the total number of occurences per play and assigned the max play to that specfic instance. If there is a tie in the maximum, it randomly chooses amongst the highest. 

```python
def get_play(row):
    name = row['Speaker']
    sentence = row['Dialogue']
    
    p = []
    q = []
    for lists_of_names in names['Name']:
        #print(lists_of_names)
        s=0
        t=0
        for n in lists_of_names:
            #print(n)
            if n in sentence:
                s+=1
            if n in name:
                t+=1
        p.append(s)
        q.append(t)

    tmp = names.copy()
    tmp['name_count'] = p
    tmp['dialogue_count'] = q
    tmp['count'] = tmp['name_count'] + tmp['dialogue_count']
    
    play = tmp[tmp['count'] == max(tmp['count'])].sample(1)['Play'].item()
    return play
```

```python
## if name in shakespeare 
def get_play_bow(row):
    name = row['Speaker']
    l = []
    for value in df['bag_of_words']:
        if name in value:
            l.append(True)
        else:
            l.append(False)

    tmp = df.copy()
    tmp['test'] = l
    tmp = tmp.groupby('Play').sum('test').reset_index()
    
    play_by_name = tmp[tmp['test'] == max(tmp['test'])].sample(1)['Play'].item()
    
    return play
```

```python
shakespeare['play'] = shakespeare.apply(lambda row: get_play(row), axis=1)
shakespeare['play_bow'] = shakespeare.apply(lambda row: get_play_bow(row), axis=1)

shakespeare[['Speaker', 'Dialogue', 'play', 'play_bow']]
```

##### Unpacking/ Example of function
We're going to unpack this a bit. We'll use the following speaker and piece of dialogue as an example:

```python
name = 'antonio'
sentence = "noble sebastian, thou let'st thy fortune sleep--die, rather; wink'st whiles thou art waking."
```

Below in `names` we have a a dataframe with all of the shakeperean plays and a column with a bag of words of all of the involved names. 

```python
names[['Play', 'Name']].sample(5)
```

Essentially, we will loop through each play and all names mentioned in that particular play. We will count and aggregate the number of names that appear in the `shakespeare` text - both in the name and in the dialogue. 

```python
p = []
q = []

for lists_of_names in names['Name']:
    s=0
    t=0
    
    for n in lists_of_names:
        if n in sentence:
            s+=1
        if n in name:
            t+=1
            
    p.append(s)
    q.append(t)

tmp = names.copy()
tmp['name_count'] = p
tmp['dialogue_count'] = q
tmp['total_count'] = tmp['name_count'] + tmp['dialogue_count']
tmp
```

Above we can see the counts of name mentions in our `name`, and `text` example. The column `total_counts` is the aggregate of mentions in both. We will select the max of this aggregate (if there is a tie, we will pick randomly) as the most likely play for the dialogue to be from.  

```python
play = tmp[tmp['count'] == max(tmp['count'])].sample(1)['Play'].item()
play
```

In this case, the function would assign `Twelth Night` as the likely play (which makes sense because Antonio and Sebastian are both characters in this play!

<!-- #region -->
# Part 3

Pick one or more of the techniques described in this chapter:

- keyword frequency
- entity relationships
- markov language model
- bag-of-words, TF-IDF
- semantic embedding


Make a case for a technique to measure how important or interesting a speaker is. The measure does not have to be both important and interesting, and you are welcome to come up with another term that represents "useful content", or tells a story (happiest speaker, worst speaker, etc.)

Whatever you choose, you must:

1. document how your technique was applied
2. describe why you believe the technique is a valid approximation or exploration of how important, interesting, etc., a speaker is.
3. list some possible weaknesses of your method, or ways you expect your assumptions could be violated within the text.
<!-- #endregion -->

##### Heap's law!

```python
from nltk.tokenize import sent_tokenize
from nltk.tokenize import RegexpTokenizer
sentences = sent_tokenize(line)
tokenizer = RegexpTokenizer('[A-Za-z]\w+')
```

```python
vocab_d={}
words=[]
heap=[]
for sentence in shakespeare['Dialogue']:
    words=words+[tokenizer.tokenize(sentence)]

## loop thorugh list of words
for i in words:
    uniq=0
    for word in i:
        if word not in vocab_d:
            vocab_d[word]=1
            uniq=uniq+1
        else:
            vocab_d[word]=vocab_d[word]+1
    heap=heap+[[len(i),uniq]]

## look at unique vs total counts, map onto V and N
V=[0]
N=[0]
for i in range(len(heap)):
    V=V+[V[i]+heap[i][1]]
    N=N+[N[i]+heap[i][0]]
```

```python
#Plot of Heaps Law
from scipy.optimize import curve_fit
def func(N,k,B):
    return k*(N**B)
popt, pcov = curve_fit(func, N, V)
k=popt[0]
B=popt[1]
print("Heaps Law : |V|=k(N^B)\nvalue of k = %d"%k)
print("value of B = %d"%B)

import matplotlib.pyplot as heaps
heaps.plot(N, func(N, *popt), 'r-',label='Fitted Curve')
heaps.plot(N, V, 'b-', label='Original graph for Heaps Law')
heaps.xlabel('Words in Collection')
heaps.ylabel('Words in Vocabulary')       
heaps.legend()
heaps.show()
```

##### Speaker Importance/ Complexity
We're going to ask which speaker was most "complex" using heap's law as a jumping off point.

```python
# https://stackoverflow.com/questions/37508659/group-by-and-count-distinct-words-in-pandas-dataframe
shakespeare['word_counts'] = shakespeare['Dialogue'].str.split().str.len()

#(shakespeare['Dialogue'])
shakespeare['distinct_word_counts'] = shakespeare.apply(lambda row: len(set(row['Dialogue'].split(' '))), axis = 1)
```

```python
heaps_per_speaker = shakespeare.groupby(['Speaker']).agg({'word_counts': 'sum', 'distinct_word_counts':'sum'}).reset_index()
heaps_per_speaker.sample(5)
```

Since we're doing this as a percentage, let's eliminate characters who say less than 10 words in this corpus since they will likely skew the percentages.

```python
heaps_per_speaker = heaps_per_speaker[heaps_per_speaker['word_counts'] > 10]
heaps_per_speaker['%_distinct'] = heaps_per_speaker['distinct_word_counts']/heaps_per_speaker['word_counts']
```

Let's think of complexity differently. In class notes, we've considered key words to be the opposite of stop words. So let's see which speaker has the least number of stopwords as a percentage

```python
import nltk
from nltk.corpus import stopwords
nltk.download('stopwords')
stop = stopwords.words('english')

shakespeare['text'] = shakespeare['Dialogue'].apply(lambda x: ' '.join([word for word in x.split() if word not in (stop)]))
```

```python
shakespeare['word_counts'] = shakespeare['text'].str.split().str.len()
shakespeare['distinct_word_counts'] = shakespeare.apply(lambda row: len(set(row['text'].split(' '))), axis = 1)

heaps_per_speaker = shakespeare.groupby(['Speaker']).agg({'word_counts': 'sum', 'distinct_word_counts':'sum'}).reset_index()
heaps_per_speaker.sample(5)

heaps_per_speaker = heaps_per_speaker[heaps_per_speaker['word_counts'] > 10]
heaps_per_speaker['%_distinct'] = heaps_per_speaker['distinct_word_counts']/heaps_per_speaker['word_counts']

heaps_per_speaker.sort_values('%_distinct').tail(10)
```

<!-- #region -->
##### Document how Technique was applied:

##### Results:


##### Weaknesses:

<!-- #endregion -->

# Sentiment of Plays

```python
from textblob import TextBlob

def getPolarity(text):
   return TextBlob(text).sentiment.polarity

shakespeare['text_polarity'] = shakespeare['Dialogue'].apply(getPolarity)
```

```python
shakespeare
```

```python
shakespeare.groupby('play').sum('text_polarity').reset_index()
```

# Markov's Language Generation Model

```python
text = ' '.join(shakespeare['Dialogue'])
```

```python
import markovify
model = markovify.Text(text, state_size=2, well_formed=False)
model.make_sentence(tries=500)
```
