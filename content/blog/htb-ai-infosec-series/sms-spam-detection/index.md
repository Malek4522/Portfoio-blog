+++
title = "SMS Spam Detection with Naive Bayes"
description = "A classical lexical baseline for smishing-style text detection and why it remains relevant."
authors = ["MALEK452"]
date = 2026-03-07
[taxonomies]
tags = ["Security", "AI", "Phishing"]
+++
<p><i>This post is based on my solution notebook for the Hack The Box AI Red Teamer path module on applying AI in infosec. I rewrote the accompanying article to focus on the methodological logic of spam detection rather than on the mechanics of the notebook itself.</i></p>
<div class="buttons centered">
  <a href="/notebooks/htb-ai-red-teamer/sms.ipynb" download>Download notebook</a>
</div>

<h2>Problem Setting</h2>
<p>
SMS spam detection is a compact but useful proxy for <b>smishing defense</b>. Unlike long-form email, SMS content is short, sparse, urgency-heavy, and lexically repetitive. Attackers rely on constrained text budgets, which often makes their language statistically easier to model than more elaborate phishing campaigns.
</p>
<p>
That characteristic makes SMS an excellent setting for classical text classification. If a method cannot perform well on short, high-signal lure text, it is unlikely to hold up in more complex messaging environments.
</p>

<h2>Why a Classical Model Is Appropriate Here</h2>
<p>
The notebook uses <b>Multinomial Naive Bayes</b> with unigram and bigram features. This is not a compromise born of simplicity alone; it is also a methodologically sensible decision.
</p>
<p>
Naive Bayes performs well when predictive signal is concentrated in sparse lexical cues. Spam and smishing messages often exhibit exactly that structure: "free", "win", "urgent", "claim", "verify", monetary references, and reward-oriented phrasing. Because many of these indicators are explicit rather than deeply contextual, a lightweight probabilistic model can remain highly competitive as a baseline.
</p>
<p>
In other words, the model is chosen because the statistical structure of the problem justifies it, not merely because it is easy to implement.
</p>

<h2>Dataset and Data Quality</h2>
<p>
The experiment uses the <b>UCI SMS Spam Collection</b>. The raw dataset contains <b>5,572</b> messages, no missing values, and <b>403</b> duplicate entries, which are removed before modeling. That leaves an effective working corpus of <b>5,169</b> rows.
</p>
<p>
Deduplication matters more than it might seem. In spam datasets, repeated templates can artificially inflate model confidence and make evaluation look stronger than it really is. Removing duplicates therefore improves the integrity of the experiment by reducing the chance that a classifier simply memorizes recurring lure variants.
</p>
<p>
The label distribution is also imbalanced toward benign <code>ham</code> messages, which mirrors real-world defensive settings. A good spam detector should therefore be judged on class-sensitive metrics rather than raw accuracy alone.
</p>

<h2>Preprocessing Strategy</h2>
<p>
Messages are lowercased, stripped of most punctuation and digits, tokenized, filtered for English stop words, stemmed with Porter stemming, and then rejoined into normalized strings.
</p>
<p>
This preprocessing pipeline reflects a clear modeling philosophy: reduce superficial lexical variation so that semantically similar lure phrases collapse toward the same feature representation. For example, stemming helps normalize families of words that differ morphologically but carry similar spam intent.
</p>
<p>
The notebook preserves <code>$</code> and <code>!</code>, which is a good choice for a spam problem. Financial cues and emphatic punctuation are not noise here; they are part of the signal.
</p>
<p>
There is, however, a tradeoff. Removing numbers may weaken sensitivity to phone numbers, short codes, or numeric prize language. For a stricter smishing study, I would test whether number-preserving normalization improves performance on lure-heavy messages containing transaction amounts, OTP prompts, or callback instructions.
</p>

<h2>Feature Design</h2>
<p>
The vectorizer uses:
</p>
<pre>ngram_range = (1, 2)
min_df = 1
max_df = 0.9</pre>
<p>
This configuration is well chosen for short text. Unigrams capture the dominant lexical cues, while bigrams recover short phrase structure such as "claim now", "free entry", or "call later". The <code>max_df</code> threshold suppresses extremely common terms that are unlikely to be discriminative across classes.
</p>
<p>
For this problem, bag-of-words features are not a weakness; they are aligned with the threat model. Smishing campaigns frequently optimize for direct lexical pressure rather than subtle semantics.
</p>

<h2>Model Selection</h2>
<p>
Rather than fixing the Naive Bayes smoothing term by intuition, the notebook performs a grid search over:
</p>
<pre>alpha = [0.01, 0.1, 0.15, 0.2, 0.25, 0.5, 0.75, 1.0]</pre>
<p>
with <b>5-fold cross-validation</b> and <b>F1-score</b> as the optimization metric. The best parameter is:
</p>
<pre>{'classifier__alpha': 0.25}</pre>
<p>
This is a sound choice. F1-score is more appropriate than plain accuracy in imbalanced security classification because it captures the tension between missing malicious messages and over-flagging benign ones. Selecting hyperparameters by F1 therefore better reflects operational cost than maximizing raw correctness.
</p>

<h2>Inference Behavior</h2>
<p>
The notebook evaluates several synthetic example messages, including reward lures, account-compromise bait, and benign social coordination text. The predictions are directionally correct: promotional prize messages and urgent verification prompts score as spam, while everyday reminders and lunch coordination messages score as non-spam.
</p>
<p>
That is exactly what a well-behaved lexical baseline should do. It responds strongly to high-signal lure language without needing deep contextual reasoning.
</p>

<h2>Why This Baseline Is Useful</h2>
<p>
Classical text models remain operationally attractive for anti-phishing pipelines because they are fast, interpretable, and cheap to deploy. A Naive Bayes model with transparent vocabulary features can be inspected by defenders, retrained quickly, and integrated as one signal among others such as sender reputation, URL features, device telemetry, or mobile-app context.
</p>
<p>
For many real systems, that is exactly the right role: not a universal phishing oracle, but a <b>high-speed lexical filter</b> that feeds a larger decision pipeline.
</p>

<h2>Limitations</h2>
<p>
The notebook does not present a formal held-out test-set evaluation after model selection; it relies on cross-validation and example inference. That is acceptable for an educational baseline but not sufficient for a paper-style performance claim. A stronger study would include a strict train-test split, confusion matrix analysis, class-wise precision and recall, calibration behavior, and robustness against deliberate obfuscation.
</p>
<p>
The model is also vulnerable to adversarial wording changes. Character substitutions, multilingual bait, URL obfuscation, benign-text padding, and stylistic drift can all reduce the effectiveness of simple lexical features. Those are not reasons to reject the method; they are reasons to place it appropriately in a layered defense.
</p>

<h2>Conclusion</h2>
<p>
This experiment is a strong example of methodological fit. Naive Bayes is chosen because SMS spam is sparse and lexically explicit. Bigrams are included because short phrase structure matters. F1-based model selection is used because class-sensitive error matters more than raw accuracy. As a result, the notebook produces a credible smishing-oriented baseline that is lightweight, explainable, and operationally realistic as an early filtering component.
</p>
