+++
title = "Sentiment Analysis with an LSTM Pipeline"
description = "A sequence-modeling baseline and what it teaches about security-adjacent NLP design."
authors = ["MALEK452"]
date = 2026-03-07
[taxonomies]
tags = ["Security", "AI", "NLP"]
+++
<p><i>This post is based on my solution notebook for the Hack The Box AI Red Teamer path module on applying AI in infosec. The underlying dataset is not security-specific, but the pipeline is relevant because it demonstrates a principled sequence-modeling workflow that can be transferred to security text tasks.</i></p>
<div class="buttons centered">
  <a href="/notebooks/htb-ai-red-teamer/sentiment.ipynb" download>Download notebook</a>
</div>

<h2>Why This Task Still Matters in an Infosec Context</h2>
<p>
At first glance, sentiment classification on movie reviews appears orthogonal to security. It is not. What matters here is not the target label but the <b>NLP design pattern</b>: raw-text normalization, vocabulary management, sequence construction, recurrent modeling, and held-out evaluation. These are the same building blocks one would use for phishing-lure classification, ticket triage, analyst note tagging, or threat-intelligence text processing.
</p>
<p>
The notebook is therefore best interpreted as a <b>transferable methodological template</b> rather than as a domain-specific security detector.
</p>

<h2>Representation Choice</h2>
<p>
The first important decision is to treat text as an ordered sequence rather than a bag of independent terms. That is why the notebook uses an <b>LSTM</b> instead of a purely count-based classifier.
</p>
<p>
This choice has a defensible motivation. Sentiment often depends on compositional structure, negation, and local temporal dependency. The phrase "not good" should not be interpreted the same way as "good", and simple lexical frequency models can struggle with that distinction. More broadly, many security-adjacent text problems also contain order-dependent meaning, especially in user reports, lure language, and free-form narratives.
</p>
<p>
An LSTM is therefore a reasonable baseline whenever the goal is to capture short- and medium-range contextual dependencies without jumping immediately to transformer architectures.
</p>

<h2>Preprocessing Philosophy</h2>
<p>
The notebook cleans the corpus by stripping HTML, lowercasing, collapsing whitespace, removing most non-alphanumeric symbols, tokenizing with NLTK, and removing English stop words. Importantly, it preserves a small set of punctuation symbols such as <code>!</code>, <code>$</code>, and <code>?</code>.
</p>
<p>
That is a subtle but meaningful design choice. In informal online text, punctuation often carries emotional or persuasive force. In security-related corpora, those same symbols can correlate with urgency, coercion, or fraud. A preprocessing pipeline that deletes all such markers may inadvertently remove useful discriminative signal.
</p>
<p>
The tradeoff is that stop-word removal can weaken sequence semantics in some contexts. For bag-of-words models it is common and often beneficial. For sequence models, the case is less obvious because function words sometimes help preserve relational meaning. This notebook keeps the pipeline simple, but that choice would deserve ablation testing in a more formal study.
</p>

<h2>Sequence Construction</h2>
<p>
The text is tokenized with a Keras tokenizer capped at a vocabulary of <b>20,000</b> words and an explicit <code>&lt;OOV&gt;</code> token. Sequences are padded to a fixed length of <b>200</b>.
</p>
<p>
Both decisions are sensible compromises. Vocabulary truncation prevents the model from overfitting to extremely rare terms and keeps the embedding layer tractable. Fixed-length padding provides computational regularity and allows batching, while the out-of-vocabulary token gives the model a principled fallback for unseen text at inference time.
</p>
<p>
In research terms, this is a controlled capacity-management strategy: bound the lexical space, bound the sequence length, and let the model learn under those constraints.
</p>

<h2>Architecture Choice</h2>
<p>
The model architecture is:
</p>
<pre>Embedding(20000, 128)
LSTM(64)
Dropout(0.5)
Dense(1, activation="sigmoid")</pre>
<p>
Each layer has a clear role. The embedding layer converts sparse token identities into dense vector representations. The LSTM models sequential dependency. The dropout layer regularizes the representation before classification. The sigmoid output is appropriate for binary prediction.
</p>
<p>
This is not a state-of-the-art architecture, and that is precisely why it is useful. It is compact enough that the modeling assumptions remain legible. If a much more complex system is later introduced, this baseline provides a clean reference point for judging whether the added complexity is actually buying meaningful performance.
</p>

<h2>Optimization and Learning Dynamics</h2>
<p>
Training uses the Adam optimizer, binary cross-entropy loss, a batch size of <b>32</b>, and <b>5 epochs</b>. The model is evaluated on a validation set split from the original training data and then on a separate test set.
</p>
<p>
The learning curve is instructive. The first three epochs remain weak, then performance improves sharply:
</p>
<pre>Epoch 1: accuracy 0.5122, val_accuracy 0.5320
Epoch 2: accuracy 0.5476, val_accuracy 0.5278
Epoch 3: accuracy 0.5653, val_accuracy 0.5408
Epoch 4: accuracy 0.8023, val_accuracy 0.8542
Epoch 5: accuracy 0.9196, val_accuracy 0.8750</pre>
<p>
This pattern is consistent with sequence models that need several epochs before the embedding space and recurrent state dynamics begin to align with the task signal. The later jump suggests the network eventually learns more meaningful token interactions rather than merely surface-level frequency patterns.
</p>

<h2>Evaluation</h2>
<p>
On the held-out test set, the model reports:
</p>
<pre>Loss:     0.3676
Accuracy: 0.8603</pre>
<p>
An <b>86.03%</b> test accuracy is respectable for a compact baseline, but the more important conclusion is interpretive rather than numerical. The architecture is strong enough to capture sequential text structure, yet still simple enough that there is substantial room for improvement through better preprocessing decisions, pretrained embeddings, bidirectional recurrence, or modern transformer baselines.
</p>

<h2>What This Teaches for Security NLP</h2>
<p>
The real value of this exercise lies in its transferability. Security text is often noisy, inconsistent, and partially structured. Whether the task is classifying phishing emails, labeling incident tickets, detecting coercive language in social engineering, or filtering open-source chatter, the same questions recur:
</p>
<ul>
  <li>What information should preprocessing preserve?</li>
  <li>How large should the vocabulary be?</li>
  <li>How much sequence context is enough?</li>
  <li>Is order-dependent modeling worth the complexity over lexical baselines?</li>
</ul>
<p>
This notebook provides a reasonable answer set for a baseline sequence model.
</p>

<h2>Limitations</h2>
<p>
The dataset is not security-native, so task transfer is conceptual rather than empirical. Any serious security application would require domain data, label design aligned with analyst workflows, and evaluation on realistic class distributions.
</p>
<p>
There is also an artifact-management caveat. The notebook saves both a Keras model file and a <code>joblib</code> dump. In practice, the native Keras serialization path is the more appropriate long-term artifact, and the tokenizer should be versioned alongside the model.
</p>
<p>
Finally, by 2026, transformer models are the default comparison point for many text tasks. That does not invalidate the LSTM baseline, but it does mean the architecture should be understood as a deliberate simplicity choice rather than a modern upper bound.
</p>

<h2>Conclusion</h2>
<p>
This experiment is valuable because it makes its assumptions visible. The LSTM is chosen to model sequence dependence, the vocabulary cap is chosen to control capacity, padding is chosen to stabilize batching, and the final accuracy shows that a compact recurrent model can still learn meaningful textual structure. As a research-style baseline for security-adjacent NLP, that is exactly the kind of result worth documenting.
</p>
