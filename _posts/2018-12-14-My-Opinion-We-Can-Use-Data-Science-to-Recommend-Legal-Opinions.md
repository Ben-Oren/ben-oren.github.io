
# My Opinion: We Can Use Data Science to Recommend Legal Opinions

### The Set-Up

Lawyers are people too (shocking, I know).  They make mistakes (even if they won't admit it).  And when they're searching for legal opinions to build arguments, the dominant tools to find those opinions are dependent on lawyers to adequately use them.  

Keyword searches and drilling down through citations are dependent on the skill of the lawyer doing them.

Even if a lawyer is a Westlaw ninja, they don't know what they don't know: they may miss a pertitnent but not obvious precedent opinion because it falls outside the bounds of their search parameters

Enter: the recommender

### The Recommender

I built a recommendation system to suggest legal opinions.  Give it an opinion and it will give back a ranked list of opinions that are most similar.  

"Most similar" means "has the most words and citations in common".  

Let's get into the nitty-gritty.

### The Nitty-Gritty

#### Data sharing is data caring

I acquired data through the caselaw access project.  People not granted access (i.e. us peons not affiliated with giant law firms or research institutions) are limited to 500 search queries a day.  Natural language processing techniques work best with large amounts of data, and in the short time available to pick up data 500 queries wouldn't cut it.

HOWEVAH.  Illinois and Arkansas, in their wisdom, have released the entirety of their state court opinions at every level - district, appellate and supreme - to the public. 

I acquired the entirety of the set of Illinois opinions in a Mongodb database, and extracted those opinions that lawyers would be interested in.  (In other words, I filtered out short insubstantial concurring and dissenting opinions that are essentially 'I agree / disagree with the majority'.  

#### Cleaning the data with holy fire

The data was relatively well-behaved; the caselaw access project does a good job of OCRing legal opinions, and there weren't weird artifacts in the texts (even from the 1800s).  

Out of 182,000 cases there were a couple thousand from 1890-1920, and 1921 was the first year in which the number of opinions in modern years was sustained.  After reading a sample of the 1890-1920 opinions and finding them materially different than modern ones, I excluded them.

I created a dataset with each opinion as an observation, and the judges, years, the opinion's citation, and the opinion text as variables.

After that, I extracted a list of citations for each case using a package called LexPredict.  I turned each case into a dummy variable, so that the dataset now looked like a list of cases with their citations as variables.

Then it was time to get into the NLP processing steps.

#### Tokens and stems and TF-IDF: the weird world of NLP 

In order to calculate similarties among opinions, their words had to be broken down into individual units whose commanlaites across datasets can be tracked.  Think of this as feature extraction: getting a list of features for each text.  A popular way to do so is by vectorizing the words: essentially creating dummy variables of each word and so that each words becomes a feature in a document-word matrix.

HOWEVAH.  Common grammatical words like 'of', 'the' and 'to' don't provide any information about document similarity since they are literally in every document, and as such are a waste of processing time to include and would through off the results.  Accordingly, they were excluded from the vectorization process.  

Another technique that's popular to vectorize words is to run them through a process that replaces each word dummy variable column - as series of 1s and 0s - with a system of weights that reflects how frequent or important a word is to the document.  Yes, the infamous TF-IDF technique, savior to wise men and fools alike.  (And me as well.)  

At this stage,  I had two document-word matrices - a TF-IDF version and a dummy variable version - to run through topic modeling techniques.  In addition, I created another matrix of each version with 'stemmed' words.  Stemming is a process that tries to bucket similar words across grammatical instances, so that words like 'lay' and 'laid', 'saw' and 'seen' are grouped respectively in broader buckets like 'lie' and 'see'.  This can be thought of as a form of feature or dimensionality reduction. 

### Topic Modeling: or, Viktor Frankl should have searched for meaning with non-negative matrix factorization 

The most important part of the process I went through to group together texts was topic modeling.  Topic modeling is a way of grouping together words that appear in similar contexts.  These groups then become the features of a document-topic matrix, and each document is assigned a weight for each topic.  This is yet another form of dimensionality reduction: replacing the tens of thousands of features that are individual words with tens of features that are topics.

#### What's the trick?

The trick is making the topics coherent.  Frequently, the list of topics will not cohere into anything resembling a intelligible structure.  As such, the information that a document is weighting that topic either more or less heavily than other topics is meaningless.  We're looking for information like "this opinion is weighting a 'police' topic heavier than other opinoins'.

A common way of trying to get more coherent topics is to try out multiple topic generating algorithms.  I tried using LSA/LDA (which use a dirichlet distrbution to probabilistically apply weights to words to get topics) and SVM and NMF (which are ways of decomposing a document-word matrix into a document-topic and topic-document matrix)

The first few iterations of topic modeling are mainly not about getting the topics coherent, but for finding the words that are too common in the corpus of texts to be useful splits but which are not obvious to exclude at the step where words like 'to', 'and' and 'of' were excluded.  Words that fell into this group were words like 'party', 'appellate', judges' names, and the like.  Because they are so common among opinoins, when included in the vectorization and topic modeling process, they created their own groups that a large number of texts fell into.  I subsequently excluded them.

I eventually settled on 20 topics with the NMF technique that were more or less coherent.  In addition to eyeballing the top 50 or so words to make sure they cohered, I also looked at the 'importance' measure assigned to each word.  This is a measure of the strentgh of each word within the topic.  I looked for topics that had a more or less even descent down importance; topics that had a top few words with an order of magnitude or more importance than the following words were deemed less coherent than topcis that didn't.  

#### Going the cosine distance 

Once the topics were found, it's a realtively straight-forward process to create a recommendation system.  The topic modeling process results in a document-topic matrix with documents as rows and their weights in each topic as columns.  Applying a similarity metric to this matrix results in a matrix similar to a covariance or correlation matrix where the relationship from each row to every other row can be read either across a row or down a column.  

Cosine distance is mathematically similar to the pearson correlation; the pearson correlation is the cosine distance with the mean removed from each observation so it's centered around 0.   One interpretation of the cosine distance is, geomterically, it's the similarity among vectors according to the angle of the vector measured from the origin.  In other words, the length of the vector is immaterial.  

Applying this to a document topic matrix, the cosine distance would count documents that are closest among multiple topics as similar, no matter how closely aligned along one particular topic they are (indeed, two opinions might have the exact same weight for one topic but have a low cosine distance if their other topic weights are dissimilar enough).  

This seems like an ideal way to measure the similarity among court opinions, each of which are always dealing with multiple issues of fact and abstract legal principles.  
