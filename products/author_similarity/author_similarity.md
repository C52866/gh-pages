---
layout: "product"
title: "Author Similarity"
metaDescription: "Similarity score between two authors or between two authors lists."
tags:
- "Author Similarity"
header:
  title: null
  desc: null
sections: []
product: "author_similarity"
permalink: "/ai/author_similarity/author_similarity_overview.html"
toc: []

---
# Author Similarity and Author List Similarity

## Summary
Similarity score between two authors or between two authors lists

## Installation
```shell
conda create --name author-similarity-env python==3.10
conda activate author-similarity-env
git clone https://github.com/Pubs-Digital-Transformation-R-D-Team/ai-author-comparison.git
cd ai-author-comparison
pip install .
```

OR 

```shell
pip install git+https://github.com/Pubs-Digital-Transformation-R-D-Team/ai-author-comparison.git
```

## Usage
[Reference file](test.py)

### Compare Authors
The function takes two input parameters, `source_author` and `target_author`, which are lists containing the author's name at index 0 and a list of texts authored by that author as subsequent elements. Each text is represented as combined title and abstract of the work.

text = title+abstract

```python
from author_similarity import AuthorSimilarity
aut_sim = AuthorSimilarity()

source_author = ['Jun Zhang',['3D printing of silk particle-reinforced chitosan hydrogel structures and their properties','However, 3D printed hydrogel scaffolds often suffer from low printing accuracy and poor mechanical properties due to their soft nature and tendency to shrink.']]

target_author = ['Jun Zhang',['3D printing of silk particle-reinforced chitosan hydrogel structures and their properties Hydrogel bioprinting is a major area of focus in the field of tissue engineering.','The printed composite hydrogel scaffolds showed no cytotoxicity, and supported adherence and growth of human fibroblasts and keratinocytes cells.']]
    
score = aut_sim.get_author_similarity(source_author, target_author)
```

The similarity score is a float value between 0 and 1, where higher values indicate greater similarity between author pair.
When using a threshold value of 0.59, we achieve the highest accuracy, which means that similarity scores below 0.59 can be considered as 'No Match' between authors. On the other hand, similarity scores greater than or equal to 0.59 can be confidently labeled as a 'Match'.

### Compare Authors Lists

Provide `source_author_dict` and `target_author_dict` such that map contains author name as key and value as list of texts that author has authored.

text = title+abstract

```python
from author_similarity import AuthorSimilarity
aut_sim = AuthorSimilarity()

source_author_dict = {'Jun Zhang':['3D printing of silk particle-reinforced chitosan hydrogel structures and their properties.','However, 3D printed hydrogel scaffolds often suffer from low printing accuracy and poor mechanical properties due to their soft nature and tendency to shrink.'],
'Benjamin James Allardyce':['Plasticised silk membranes made from formic acid are ductile, transparent and degradation-resistant	Regenerated silk fibroin membranes tend to be brittle when dry.','They also showed good biocompatibility and supported the adhesion and migration of human tympanic membrane keratinocytes.','Silk protein paper with in-situ synthesized silver nanoparticles'],
'Rangam Rajkhowa':['Stretchable, Self-healable, and Biocompatible Silk Fibroin Hybrid Membranes based Humidity Sensor and Human Emotion Sensor']}
        
target_author_dict = {'Jun Zhang':['3D printing of silk particle-reinforced chitosan hydrogel structures and their properties.','However, 3D printed hydrogel scaffolds often suffer from low printing accuracy and poor mechanical properties due to their soft nature and tendency to shrink.'],
'Benjamin James Allardyce':['Plasticised silk membranes made from formic acid are ductile, transparent and degradation-resistant	Regenerated silk fibroin membranes tend to be brittle when dry.','They also showed good biocompatibility and supported the adhesion and migration of human tympanic membrane keratinocytes.','Silk protein paper with in-situ synthesized silver nanoparticles'],
'Rangam Rajkhowa':['Stretchable, Self-healable, and Biocompatible Silk Fibroin Hybrid Membranes based Humidity Sensor and Human Emotion Sensor']}
                          }
score = aut_sim.get_authorlist_similarity(source_author_dict, target_author_dict)
```

The similarity score is a float value between 0 and 1, where higher values indicate greater similarity between author list pair.
When using a threshold value of 0.63, we achieve the highest accuracy, which means that similarity scores below 0.63 can be considered as 'No Match' between author lists. On the other hand, similarity scores greater than or equal to 0.63 can be confidently labeled as a 'Match'.

## Architecture
### Author Comparison
![Author Comparison](/gh-pages/assets/doc/Author_Comparison.png)

### Author List Comparison
In this comparison, scenarios where one list contains more authors than the other is addressed. To achieve this, comparison between the author list with more authors and the author list with fewer authors is done. This ensures that we obtain distinct similarity scores for the following cases:

1. Two manuscripts with exactly the same author lists.
2. Two manuscripts, where the author list for one manuscript is a subset of the other.

By handling these cases separately, obtaining the same scores for both scenarios can be avoided, providing more accurate and informative results.

![Author Lists Comparison](/gh-pages/assets/doc/AuthorList_Comparison.png)

## References
- <https://github.com/allenai/SPECTER2_0>
- <https://github.com/seatgeek/thefuzz>
