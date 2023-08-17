---
layout: "product"
title: "Author Similarity Evaluation"
metaDescription: "We evaluated on 4 models for author list similarity:\r\n\r\nSiamese\
  \ Network trained on author’s text (abstract)\r\nSpecter\r\nSpecter 2\r\nS2AND"
tags:
- "Author Similarity"
header:
  title: null
  desc: null
sections: []
product: "author_similarity"
permalink: "/ai/author_similarity/author_similarity_evaluation.html"
toc: []

---
# Evaluating Algorithms for Author List Comparison

We evaluated on 4 models for author list similarity:
1.	Siamese Network trained on author’s text (abstract)
2.	Specter 
3.	Specter 2
4.	S2AND

## Siamese Network Fine tuned with author’s text (abstract)
1.	Generate dataset using from MSPUBS.
2.	Prepare training, test and validation dataset with 3 sentences for each author id with anchor, positive and negative example – Refer file : training_data.py A,P,N  (80-10-10 split)
3.	Train Siamese network – train_author_sbert.py

[Refer here](../evaluation/author_siamese_network)

Base accuracy for this model = 95.8%
epoch,steps,accuracy_cosinus,accuracy_manhattan,accuracy_euclidean
0,98000,0.9589267898071997,0.959493056491843,0.9593905891870028
0,-1,0.9588405015504922,0.9593636241067817,0.9594984495078873

## Evaluate 1,2 and 3

To evaluate 1., 2. and 3. below steps are followed:
- Get manuscripts submitted after Apr 2023 for 5 days since siamese network was trained on data before Apr 2023. (~2k manuscripts)
- Get authors associated with that manuscript.
- Generate author embeddings for each author by generating text embeddings for all past submissions of an author and averaging it (text – title + abstract).
- Find similarity between author embedding and manuscript text embedding (cosine similarity).
- Authors prediction – Get Top N similar authors per manuscript (N= no of authors in a manuscript) and get # of authors that match actual authors.
- Accuracy = Sum of # authors predicted correctly in top N/ Sum of # of authors in authors list

|            | SPECTER | Trained Siamese Network  | SPECTER2.0  |
| :------------- | ------ | ------ | ------ |
| Top N      | 0.740 | 0.699 |  0.776 |
| Top N+1    | 0.755 | 0.714 |  0.789 |
| Top N+2    | 0.767 | 0.725 |  0.8   |
| Top N+3    | 0.776 | 0.735 |  0.808 |
| Percentage of manuscripts with # of authors>= 1 predicted correctly| 0.984 | 0.975 | 0.988 |
| Percentage of manuscripts with # of authors>= 2 predicted correctly| 0.896 | 0.877 | 0.916 |
| Percentage of manuscripts with # of authors>= 3 predicted correctly| 0.771 | 0.735 | 0.788 |



## Evaluating S2AND and SPECTER2.0 on Author list using Controlled Data

### S2AND

There are 8 datasets used by S2AND for author name disambiguation which are - Aminer, ArnetMiner, INSPIRE, KISTI, Medline, PubMed, QIAN, SCAD-zbMATH.
It follows a 3-stage name disambiguation approach –
1.	Blocking : Records with same first initial and last name are put into same block.
2.	Pairwise Similarity Estimation : Here the features it uses for pairwise similarity are : first name, middle name, affiliation, email, co-authors, venue, year, title, abstract, references, author position, language, name counts, specter embeddings using title and abstract, journal.
3.	Agglomerative Clustering : Using the trained pairwise classifier from the previous step, distance matrix is constructed with probability that 2 records are not by same author. Here fastcluster with average linking is used for agglomerative clustering.

The reference model is live on semanticscholar.org for author name disambiguation

```bash
git clone https://github.com/allenai/S2AND.git
cd S2AND
conda create -y --name controlled-data-eval python==3.8
conda activate controlled-data-eval
pip install -r requirements.in
pip install -e .
```

```shell
conda env update -f s2and-controlled-data-eval-env.yml
```

#### Install AWS CLI

```bash
$ wget https://s3.amazonaws.com/aws-cli/awscli-bundle.zip
$ unzip awscli-bundle.zip
$ ./awscli-bundle/install -b ~/bin/aws
$ echo $PATH | grep ~/bin     // See if $PATH contains ~/bin (output will be empty if it doesn't)
$ export PATH=~/bin:$PATH     // Add ~/bin to $PATH if necessary
```

#### Download pre-trained S2AND model

```bash
aws s3 sync --no-sign-request s3://ai2-s2-research-public/s2and-release data/
```

#### Configuration

Modify the config file at data/path_config.json. This file should look like this

```python
{
    "main_data_dir": "absolute path to wherever you downloaded the data to",
    "internal_data_dir": ""
}

```

#### Inferencing

##### Generate S2AND dataset with controlled data

Reference file : create_dataset_s2and.py

3 files needs to be generated : papers.json, signatures.json, specter.pickle

**acs_papers.json** - File which consists of manuscript and dois with it's details - title, abstract, journal name, year of publication/submission, author details-author name and position in a paper.

**acs_signatures.json** - Here signature refers to author signature appeared in a paper, which is unique per author per paper. Here author details are provided like author id, paper id, author signature- unique per author per paper, author block- initial+last name, author psoition in a paper, first name, middle name, last name, suffix, affiliations, email.
Note- Author position, email details are not provided by Open Alex

**acs_papers_specter.pickle** - Here each paper's title+abstract is stored as specter embedding.

##### Use generated S2AND dataset for prediction

Reference file : s2and_prediction.py
https://github.com/allenai/S2AND/tree/main

```python

import pickle

with open("production_model.pkl", "rb") as _pkl_file:
    clusterer = joblib.load(_pkl_file)
    clusterer.use_cache = False
    clusterer.use_default_constraints_as_supervision=False

dataset_name = "acs"

anddata = ANDData(
    signatures=join(parent_dir, f"{dataset_name}_signatures.json"),
    papers=join(parent_dir, f"{dataset_name}_papers.json"),
    specter_embeddings=join(parent_dir, f"{dataset_name}_papers_specter.pickle"),
    name="ACS Controlled Data",
    mode="inference",
    block_type="s2",
)

pred_clusters, pred_distance_matrices = clusterer.predict(anddata.get_blocks(), anddata)

```

![S2AND Author Data Preparation Controlled Data](../doc/Data_Preparation_S2AND.png)

##### Predicting for ACS Controlled Dataset

Save AND data blocks and distance matrix generated above.
For each record in controlled data, do below steps :

a) Get authors for source and target manuscripts
     If source or target authors are not available for manuscript, set score=0
     Source authors list, Target authors list; size of source author list > size of target      	 author list 
     
b) Get author signature by combining the author id and manuscript (as stored in signatures.json file.
  
c) Get the block of each author in source authors and target authors list using it's signature. If no block is found, ignore that author and go to step b.
  
d) Compare block of source author and target author, if it matches, we can get the distance score from distance matrix of that block generated by S2AND prediction. Hence, similarity=1-distance. Else set similarity score = 0 
  
e) Get score with highest similarity in target author list for each source author list.
  
f) Take an average of scores generated in e, which is author list similarity score between source and target authors.

![S2AND Author Evaluation Using Controlled Data](../doc/author_eval_S2AND.png)

### SPECTER2.0

Given the combination of title and abstract of a scientific papers, the model can be used to generate effective embeddings.

#### Usage

```bash
conda activate controlled-data-eval
```

```python
from sentence_transformers import SentenceTransformer, util
specter2_model = SentenceTransformer('allenai/specter2')
specter2_model.encode(['We introduce a new language representation model called BERT','The dominant sequence transduction models are based on complex recurrent or convolutional neural networks'])
```

#### Dataset preparation

1. Get manuscripts from controlled data sheets
2. Get authors for each manuscript in controlled data
3. Get all unique authors generated from step 2.
4. Get set of all manuscripts authored by each of the unique authors generated from step 3
5. For each author - generate SPECTER2 embeddings for all authored text (title+abstract)
6. Generate author embedding by averaging SPECTER2 embeddings of each authored manuscripts

Reference file : prepare_dataset_specter2.py
Output : {author_id:embedding}

![SPECTER2 Author Evaluation Using Controlled Data](../doc/Data_Preparation_SPECTER2.png)

#### Inferencing

To calculate similarity between 2 author ids, we can calculate cosine similarity between their embeddings.
To calculate author list similarity, for each record in controlled data, follow steps below :

a) Get authors for source and target manuscripts
     If source or target authors are not available for manuscript, set score=0
     Source authors list, Target authors list; size of source author list > size of target author list
     
b) For each author in source list, get embedding. Ignore author if embedding is not present. Do the same for target author list.

c) Use the list of authors generated from step b for source and target author list.
    It considers the authors of the manuscript with more authors as the source author list and the authors of the manuscript with fewer authors as the target manuscript.

d) Get cosine similarity between source author list embedding and target author list embedding.

e) For each author in source manuscript, get cosine similarity of the author with each author in target manuscript. Similarity between 2 authors can be calculated by using cosine similarity score and fuzzy name matching score. Similarity score = Cosine similarity * fuzzy name matching score

f) Get the maximum matched score author for each of source author

g) Take average of max matched scores for all source authors. This is the final similarity score between author lists.

![SPECTER2 Author Evaluation Using Controlled Data](../doc/author_eval-SPECTER2.png)

### Evaluation Using Highwire Controlled Dataset

#### Sample Dataset Used for Author List Pair :
1.	Positive class : Pick random 71 samples of ‘Exact Match’ where author list similarity score < 0.98 for both the algorithms from previous run.
2.	Negative class : 71 samples of ‘No Match’ where title score > 0.4 (already provided)
 
Prediction : If author list similarity score >= threshold value, it is predicted as positive class (Exact Match), else negative class (No Match).
 
SPECTER2 + Fuzzy Match: Got maximum accuracy = 99.3% for threshold value of 0.63, 
S2AND : Got maximum accuracy = 99.3% for threshold values between 0.2 to 0.49, 
 
Accuracy scores for each threshold values at [Accuracy Scores](../doc/highwire_with_scores_both.csv)

#### Sample Dataset Used for Author Pair :
1.	Positive class : Consider rows that exhibit an exact match (with a high similarity score obtained from author list similarity score, approximately 100). For these matched rows, we identify instances where authors match, and these samples are utilized for author similarity analysis. Specifically, we compared [A1, A2, A3] with [B1, B2, B3].
In this context, let's take the example of A1-B1, which has the highest similarity score. We use this pair as a sample representing an exact author match. Even though A1 and B1 are the same authors, their names and texts are obtained from different datasets, namely, Open Alex and ACS submissions. Similarly, we can use A2-B2 and A3-B3 as additional samples for author similarity analysis.
2.	Negative class : Considered authors who co-authored multiple papers (>=10). In such cases, their names may not match exactly, but the content of their work will exhibit similarities. These co-authors can be utilized as negative samples for author similarity comparison.

Prediction : If author similarity score >= threshold value, it is predicted as positive class (Exact Match), else negative class (No Match).

The dataset consisted of approximately 700 samples, with an equal proportion of both types of author pairs. During the evaluation, maximum accuracy of 99.14% was achieved when using threshold values of 0.59 and 0.63 using SPECTER2 + Fuzzy Match Algorithm.

We have decided to adopt a threshold value of 0.59 as our midpoint. This choice is based on the objective of reducing false negatives, thus ensuring that fewer instances of actual author matches are incorrectly classified as non-matches.

[Reference File](author-comparison.py)

## Future R&D

![Future R&D](../doc/author_embedding_future.png)

## References

- <https://github.com/allenai/S2AND/tree/main>
- <https://blog.allenai.org/s2and-an-improved-author-disambiguation-system-for-semantic-scholar-d09380da30e6>
- <https://github.com/allenai/SPECTER2_0>
- <https://github.com/seatgeek/thefuzz>

