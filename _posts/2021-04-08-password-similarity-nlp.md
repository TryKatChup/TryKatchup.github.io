---
layout: post
section-type: post
title: Password Similarity Detection Using Deep Neural Networks
category: deep-neural-networks
tags: [ 'password-similarity', 'NLP', 'privacy', 'word-embedding']
---

Password Similarity Detection Using Deep Neural Networks. This project was the case study of my bachelor's thesis.

Source code can be found [here](https://github.com/TryKatChup/password-similarity-nlp)
<div align="center">
 <img src="/assets/logo.png">
</div>

---
# Introduction

Nowadays people tend to create more profiles or change password of their current profiles for security reasons. Existent passwords and literature-based words have a great impact on the candidate password. This could be a risk for the user privacy. For example, an user has the password `mum77` and he/she wants to create a new account for a different website. A candidate password could be `mommy1977`, which is a variation of `mum77` and it more risky if an attacker has discovered the first password in a leak. 

The purpose of this project is to give a feedback about password similarity between the new password and the old one using Deep Neural Networks. 
As a reference, a [scientific article published at IEEE Symposium on Security and Privacy 2019](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=8835247was) chosen. Then the entire architecture was reimplemented and improved, and a comparison between the obtained results and the case study was made.

---
# Data pre-processing
**File:** [`preparing_dataset.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/preparing_dataset.py)

First of all I used a compilation of password leaks containing 1.4 billion email-password pairs from the Deep Web. Further operations were applied on the dataset:
<td style="text-align: left;">
<ul>
    <li>Passwords longer than 30 characters or shorter than 4 removal.</li>
    <li>Non ASCII printable characters in password removal.</li>
    <li>Bot (which are recognisable by the same mail used more than 100 times) removal.</li>
    <li>HEX passwords (identified by <code>$HEX[]</code>) and <code>\x</code> removal.</li>
    <li>HTML char set removal, for example:</li>  
    <ul>
        <li><code>&gt</code></li>
        <li><code>&ge</code></li>
        <li><code>&lt</code></li>
        <li><code>&le</code></li>
        <li><code>&#</code>(HTML entity code)</li>
        <li><code>amp</code></li>
    </ul>
    <li>Due to the impossibility of similarity detection, accounts with less than 2 password were removed.</li> 
</ul>
</td>
After that, two dataset were build:

<ul>
    <li>The first one, according with Bijeeta et alii, contains all the passwords in key-presses format (using the <code>word2keypress</code> package).</li>
    <li>The second one contains all the passwords as they are.</li>
</ul>   
    
The filtered datasets were saved in two `.csv` files in this format:

<ul>
    <li>in the first dataset: <code>sample@gmail.com:["'97314348'", "'voyager&lt;s&gt;1'"]</code></li>
    <li>in the second dataset: <code>sample@gmail.com:["'97314348'", "'voyager!'"]</code></li>
</ul>

## Word2keypress

Every password in the first dataset was translated in a keypress sequence on an ANSI american keyboard:

<ul>
    <li>Every capital letter was represented by <code>&lt;s&gt;</code> (the <code>SHIFT</code> key) before the lowercase version.
        <br>e.g. <code>Hello -> &lt;s&gt;hello</code></li>
    <li>If there is a sequence of consecutive capital letters, followed by lowercase letters, the <code>&lt;c&gt;</code> tag (the <code>CAPS LOCK</code> key) is inserted <i>before</i> and <i>after</i> the sequence, which will be represented by lowercase letters. 
  e.g. <br><code>Password -> &lt;c&gt;pass&lt;c&gt;word</code></li>
    <li>If a sequence of capital letters ends at the end of the word, the <code>&lt;c&gt;</code> tag wil be placed before the sequence.
 <br>e.g.<br> <code>PASSWORD -> &lt;c&gt;password</code>
  <br><code>passWORD -> pass&lt;c&gt;word</code></li>
    <li>If a password contains ASCII 128 special characters, the <code>&lt;s&gt;</code> tag will be placed before the special character, which is translated as <code>SHIFT + &lt;key for the specific character&gt;</code>
        <br>e.g. <br><code>PASSWORD! -> &lt;c&gt;password&lt;s&gt;1</code>
        <br><code>Hello@!! -> &lt;s&gt;hello&lt;s&gt;2&lt;s&gt;1&lt;s&gt;1</code></li>
</ul>

<div align="center">
    <img style="max-height: 256px" src="/assets/US_keyboard_layout.png">
</div>


## Splitting dataset
**File:** [`split_dataset.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/split_dataset.py)

Both datasets are splitted in training set (which is 90% of the original dataset) and test set (the remaining 10% of the original dataset). 

---
# Training FastText
**File:** [`FastText_training.ipynb`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/FastText_training.ipynb) **and** [`PasswordRetriever.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/PasswordRetriever.py)

In this file FastText will be trained based on the given training set.
In order to understand better `Training dataset.py` and FastText, Word2Vec is briefly introduced.

## Word2Vec
Word2Vec is a set of architectural and optimization models which learn word embeddings from a large dataset, using deep neural networks.
A model trained with Word2Vec can detect similar words (based on context) thanks to _cosine similarity_.

Word2Vec is based on two architectures:

<ul>
    <li><b>CBOW</b> (continuous bag of words): the main purpose is to combine the representation of surrounding words, in order to predict the word in the middle.</li>
    <li><b>Skip-gram</b>: similar to CBOW, except for the word in the middle, which is used to predict words related to the same context.</li>
</ul>


CBOW is faster and effective with larger dataset, however, despite the greater complexity, Skip-gram is capable to find _out of dictionary_ words for smaller datasets.

<div align="center">
 <img style="max-height: 256px" src="/assets/cbow_vs_skipgram.png">
</div>


## FastText
[FastText](https://fasttext.cc/) is a open source library created by Facebook which extends Word2Vec and is capable to learn word representation and sentence classification efficiently. The training is based on password n-grams. 
N-grams of a specific word that contains `n` characters are defined as follow:
<img src="/assets/nngram_formula.png" >

For example, the ngrams of the word `world`, with `n_mingram = 1` and `n_maxgram = 5` are:   
<br>
  


`world` is represented as the subset of substrings with 1 and 5 as respectively minimum and maximum length.
FastText, comparing to Word2Vec, is capable to obtain more _out of dictionary_ words, which are unknown during the training phase.

## Environment setup
In order to train the model, [`gensim.FastText`](https://radimrehurek.com/gensim/models/fasttext.html) was used.   
[`gensim`](https://radimrehurek.com/gensim/) is a open source and cross-platform python library, with multiple pre-trained word embedding models. 
`PasswordRetriever` class extracts all the passwords from the `.csv` preprocessed file.

## FastText parameters
```python
negative = 5
subsampling = 1e-3
min_count = 10
min_n = 2
max_n = 4
SIZE = 200
sg = 1
```
In this project _Skip-gram_ model (`sg = 1`) and negative sampling were used:
  

<ul>
    <li>Skip-gram approach is chosen, as the distributed representation of the input word is used to predict the context. Skip-gram model works better with subword information, so it is recommended for learning passwords and rare words in general.</li>

    <li>Negative sampling makes the training faster. Each training sample updates only a small percentage of the model weights. For larger datasets (like this case) it is recommended to set negative sampling between 2 and 5.</li>

    <li>The dimension of vectors is set to 200, in order to have train the model faster. Generally it is recommended to have <code>SIZE = 300</code></li>

    <li> The <code>subsampling</code> ignores the most frequent password (more than 1000 occurrences).</li>

    <li><code>min_count</code> represents the minimum number of occurences of a password in the training dataset.</li>

    <li><code>min_n</code> and <code>max_n</code> are the number of respectively minimum and maximum n-grams.</li>

    <li>N-grams are used in statistical natural language processing to predict a word and/or the context of a word. In this case they represent a contiguous sequence of n characters and their purpose is to give subword information.</li>

    <ul>
        <li>For example a password <code>w = qwerty</code> with <code>min_n = 4</code> and <code>m_max = 5</code>, will have the following n-grams. 
        <br> 
  <code>zw = { &lt;qwe, qwer, wert, erty, rty&gt;, &lt;qwer, qwert, werty, erty&gt; }</code></li>
   <br>
  NB &lt; and &gt; are considered as characters.
    </ul>
</ul>


## Saving the model
Finally, the trained model is saved as `.bin`.
```python
from gensim.models.fasttext import save_facebook_model
save_facebook_model(trained_model, "model_password_similarity.bin")
print("Model saved successfully.")
```

---
## More trainings

5 models were trained:


-  `word2keypress`, `epochs = 5`, `n_mingram = 1` (Bijeeta et al.);
-  `word2keypress`, `epochs = 5`, `n_mingram = 2`;
-  no `word2keypress`, `epochs = 10`, `n_mingram = 1`;
-  no `word2keypress`, `epochs = 10`, `n_mingram = 2`;
-  no `word2keypress`, `epochs = 5`, `n_mingram = 2`.

---
# Compressing the model

The trained model is 4.8 GB. There are some problems about the size:
<ul>
    <li>Too much space occupied in memory.</li>
    <li>It is harder to use the model in client/server architectures. Everyone should use this model, which can be sent as a payload. Embeddings are not reversible, and they guarantee password anonymization.</li>
</ul>

For these reasons, [`compress_fasttext`](https://pypi.org/project/compress-fasttext/) is used.
```python
import gensim
import compress_fasttext
```

In order to obtain a compressed model, without impacting significantly on performances, product quantitation and feature selection are applied.

<ul>
<li>Feature selection is the process of selecting a subset of relevant features for use in model construction or in other words, the selection of the most important features.</li>

<li>Product quantization is a particolar type of vector quantization, a lossy compression technique used in speech and image coding. A product quantizer can generate anexponentially large codebook at very low memory/time cost.</li>

</ul>
``` python
big_model = gensim.models.fasttext.load_facebook_vectors('model_password_similarity.bin')
small_model = compress_fasttext.prune_ft_freq(big_model, pq=True)
small_model.save('compressed_model')
```
The compressed model is 20MB.

---
# Evaluating the model
**File:** [`w2kp_PRGraph.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/w2kp_PRGraph.py) **and** [`No_w2kp_PRGraph.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/No_w2kp_PRGraph.py)

For the evaluation of the models, compressed versions obtained with product quantization were used. In order to measure any performance differences between the original model and the compressed version, Bijeeta et al. model is chosen, with the following features:

<ul>
<li>translation of the sequence of key pressed for the password;</li>
<li><code>min_gram = 1</code>;</li>
<li><code>max_gram = 4</code>;</li>
<li>epochs = 5.</li>
</ul>

An effective valutation of both model is based on _precision_ and _recall_. 
Not remarkable differences were observed: for this reason only the compressed version of the models are considered.

<br>
<div align="center">
  <img style="max-height: 350px" src="/assets/big_model.png">
</div>
<center>Precision and recall in the uncompressed model of Bijeeta et al.</center>
<br>
<br>
<div align="center">
  <img style="max-height: 350px" src="/assets/w2kp_nmingram=1_epochs=5.png">
</div>
<center>Precision and recall in the compressed model of Bijeeta et al.</center>

## Euristhics
For a proper evaluation the following euristhics is adopted:

<ul>
<li>comparing the password to the lowercase version;</li>
<li>comparing the password to the uppercase version;</li>
<li>comparing the password to the <code>l33t</code> code version;</li>
<li>verifing if edit distance is greater than 5.</li>
</ul>

---
## Ground truth and prediction
Ground truth is the ideal result expected and it is used in order to verify how much correct are model prediction.
The term _prediction_ refers to the model output after the training. It is applied to new data when is required to verify an event probability.

Ground truth depends on the candidate two passwords and the chosen euristhics, while prediction is obtained thanks to `gensim.similarity` function of the trained model (which is based on cosine similarity). In case of the `work2keypress` model, it is important to convert the candidate two password in key presses (`w2kp_PRGraph.py`).


## Precision and recall
**Precision** represents the number of true positive detected from true positives and false positives.
**Recall** represents the number of positive elements detected from a set of false negatives and true positives.

<br>
<div align="center">
  <img style="max-height: 500px" src="/assets/precisionrecall.png" >
</div>
In this case:

<ul>
    <li>A couple of similar passwords detected correctly represents <b>true positives</b></li>
    <li>A couple of different password detected as similar are <b>false positives</b>.</li>
    <li>A couple of similar passwords not detected as similar are <b>false negatives</b>.</li>
</ul>

In order to find out _precision_ and _recall_ it is important to re-define few concepts, according to [ground truth and prediction subchapter](#ground-truth-and-prediction):

<ul>
    <li> <b>True positives (TP)</b>: its value increments only if both prediction and ground truth have a strictly positive value.</li>
    <li> <b>False positives (FP)</b>: its value increments only if ground truth value is zero and  if prediction has a strictly positive value.</li>
    <li> <b>False negatives (FP)</b>: its value increments only if ground truth value is strictly positive and if prediction value is zero.</li>
</ul>
    
<center>
$ precision = \frac{TP}{TP + FP} $
<br>
$ recall = \frac{TP}{TP + FN} $
</center>

<br>

---
## Comparing the results of the models
In order to classify passwords, it is necessary to define a threeshold α, which it is chosen considering the best precision and recall values.

<ul>
    <li>Model with <code>word2keypress</code>, <code>n_mingram = 1</code>, <code>epochs = 5</code> (Bijeeta et al.):</li>

  <ul>
      <li>α = 0.5: precision ≈ 57%, recall ≈ 95%</li>
      <li>α = 0.6: precision ≈ 67%, recall ≈ 89%</li>
  </ul>

    <li>Model without <code>word2keypress</code>, <code>n_mingram = 2</code>, <code>epochs = 5</code>: </li>
    <ul>
        <li>α = 0.5: precision ≈ 65%, recall ≈ 95%</li>
        <li>α = 0.6: precision ≈ 77%, recall ≈ 89%</li>
    </ul>
</ul>


In this case study, it is more important an higher value of recall. In this way more similar passwords are detected.   
It is also important to not have too many false positives identified by couples of password which are different from each other but are considered similar. For this reason I have chosen an higher value of precision, comparing to Bijeeta et al. paper and α = 0.6.

<div align="center">
  <img style="max-height: 350px" src="/assets/w2kp_nmingram=1_epochs=5.png">
</div>

<center>Precision and recall with word2keypress, n_mingram = 1, epochs = 5 (worst model)</center>
<br>

<div align="center">
  <img style="max-height: 350px" src="/assets/no_w2kp_nmingram=2_epochs=5.png">
</div>

<center>Precision and recall without word2keypress, n_mingram = 2, epochs = 5 (best model).</center>


---
## Bijeeta et al. model issues
The worst performances were noticed in the Bijeeta et al. model. The main problems are:

<ul>
<li><code>word2keypress</code> library translates each character as a key press sequence. When a password is expressed in camel notation, an alternation of capital letters and lowercase letters is present. As a consequence, different passwords in camel notation will be considered similar, because of the repetition of <code>&lt;s&gt;</code> and <code>&lt;c&gt;</code>.</li>
<li>Two passwords expressed as key presses like:</li>
<ul>
    <br>
    <li><code>$1mp@t1c*</code>  which is translated as <code>&lt;s&gt;41mp&lt;s&gt;2t1c&lt;s&gt;8</code></li>
    <li><code>#wlng%p*m}</code> which is translated as <code>&lt;s&gt;3wlng&lt;s&gt;5p&lt;s&gt;8m&lt;s&gt;[</code></li>
</ul>
will be considered similar.   
<li><code>SHIFT</code> is translated as <code>&lt;s&gt;</code>: for this reason two passwords which are secure will appear similar.</li>


<li>Setting <code>n_mingram = 1</code> make the evaluation less precise than using <code>n_mingram = 2</code>.   
  In fact, in passwords, single characters in passwords do not depend on syntactic rules (unlike in english literature). There are multiple factors really different from each other in order to establish a set of rules for the position of a character in a password, so it is impossible to define how a character is placed.</li>
</ul>

---
## Graphic representation of words distance
**File:** [`visualize_embeddings.py`](https://github.com/TryKatChup/password-similarity-nlp/blob/main/visualize_embeddings.py)

To simplify the comprehension of the project topic, password similarity is represented with a 3-dimensional graphic. [`t-SNE` algorithm](https://scikit-learn.org/stable/modules/generated/sklearn.manifold.TSNE.html) is used to reduce the model dimension from 200 to 3. In the next figure it is possible to see the top 5 most similar passwords to `ipwnedyou` and `numBerOne` and their distances. 

<div align="center">
 <img style="max-height: 350px" src="/assets/3dplot.png">
</div>

## Explanation video (Italian only)
![](https://www.youtube.com/watch?v=G1sf4kpTGTk)
