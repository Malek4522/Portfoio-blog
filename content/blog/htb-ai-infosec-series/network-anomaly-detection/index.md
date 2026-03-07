+++
title = "Network Anomaly Detection with Random Forests"
description = "A structured-feature intrusion detection baseline on NSL-KDD with multi-class attack grouping."
authors = ["MALEK452"]
date = 2026-03-07
[taxonomies]
tags = ["Security", "AI", "Network"]
+++
<p><i>This post is based on my solution notebook for the Hack The Box AI Red Teamer path module on applying AI in infosec. The notebook itself is compact; the discussion below reframes it as a methodological study of attack categorization in tabular network data.</i></p>
<div class="buttons centered">
  <a href="/notebooks/htb-ai-red-teamer/network.ipynb" download>Download notebook</a>
</div>

<h2>Problem Setting</h2>
<p>
Network intrusion detection can be framed at multiple levels of granularity. The simplest formulation is binary: normal versus malicious. That framing is operationally useful for alerting, but it is incomplete for response. A volumetric denial-of-service event, a reconnaissance scan, a privilege-escalation attempt, and an unauthorized access pattern impose different investigative and containment priorities.
</p>
<p>
The notebook therefore adopts a <b>multi-class</b> formulation. Instead of predicting individual NSL-KDD attack names directly, it groups them into broader response-oriented categories:
</p>
<ul>
  <li><b>Normal</b></li>
  <li><b>DoS</b></li>
  <li><b>Probe</b></li>
  <li><b>Privilege</b></li>
  <li><b>Access</b></li>
</ul>
<p>
This design is sensible. It reduces label fragmentation while preserving distinctions that matter to defenders. In research terms, it is a compromise between coarse anomaly detection and fine-grained attack taxonomy.
</p>

<h2>Why a Tabular Model?</h2>
<p>
The NSL-KDD dataset is fundamentally a structured-feature dataset. Each record already contains engineered attributes such as byte counts, error rates, host-level statistics, and service-level frequencies. Under those conditions, tree-based ensemble methods are often a stronger baseline than deep models because the representation work has already been done.
</p>
<p>
That is why the notebook uses a <b>RandomForestClassifier</b>. Random forests are particularly well suited to this setting for four reasons:
</p>
<p>
They naturally handle nonlinear interactions without requiring feature scaling. They are robust to mixed feature distributions. They perform well when informative thresholds exist in the data. And they provide strong baseline accuracy on structured intrusion-detection corpora with limited hyperparameter tuning.
</p>
<p>
This is an example of choosing a model that matches the data modality rather than defaulting to neural architectures for their own sake.
</p>

<h2>Feature Representation</h2>
<p>
The dataset contains <b>148,517</b> rows. Categorical fields are partially encoded using one-hot vectors for <code>protocol_type</code> and <code>service</code>, then joined with <b>34</b> numeric traffic features. These numeric variables capture flow size, login behavior, fragment anomalies, connection density, and host/service error rates.
</p>
<p>
The choice of features reflects a classic IDS assumption: malicious traffic differs from benign traffic not only by payload content but also by its statistical footprint. Repeated failed connections, abnormal byte asymmetry, unexpected service concentration, or unusually high host error rates can all function as predictive signals.
</p>
<p>
There is also an instructive omission. The notebook does not encode the <code>flag</code> feature, even though it exists in the dataset and often carries meaningful connection-state information. The fact that the model still performs well suggests the remaining feature set is highly informative, but it also means the experiment is best read as a strong baseline rather than a maximally engineered version of NSL-KDD.
</p>

<h2>Label Engineering</h2>
<p>
A major methodological step is the mapping from individual attacks to grouped attack families. This is not just cosmetic label cleanup. It imposes a defender-oriented inductive structure on the dataset.
</p>
<p>
For example, grouping <code>nmap</code>, <code>portsweep</code>, and <code>satan</code> into <b>Probe</b> reflects the fact that these behaviors are distinct at the tactical level but similar in terms of reconnaissance intent. Likewise, consolidating exploit-like behaviors into <b>Privilege</b> and credential or service abuse into <b>Access</b> makes the output more interpretable from a triage perspective.
</p>
<p>
This kind of label engineering often matters as much as model selection because it determines what the model is actually being asked to learn.
</p>

<h2>Experimental Design</h2>
<p>
The data is split into train and test partitions with <b>80/20</b> separation, followed by an additional validation split carved from the training set. The model uses <code>class_weight="balanced"</code>, which is important because attack categories are unevenly represented.
</p>
<p>
That weighting is not sufficient to eliminate imbalance effects, but it is a reasonable first response. It tells the forest not to optimize purely for the dominant classes, which in this dataset would otherwise lead to inflated global accuracy and weak minority-class recall.
</p>

<h2>Why the Results Look Better Than They Are</h2>
<p>
The validation metrics are extremely high:
</p>
<pre>Accuracy:  0.9947
Precision: 0.9946
Recall:    0.9947
F1-score:  0.9945</pre>
<p>
The test metrics remain similarly strong:
</p>
<pre>Accuracy:  0.9950
Precision: 0.9949
Recall:    0.9950
F1-score:  0.9949</pre>
<p>
A superficial reading would conclude that the detection problem is essentially solved. That conclusion would be incorrect.
</p>
<p>
The detailed class report reveals the real story. <b>Normal</b>, <b>DoS</b>, and <b>Probe</b> are classified almost perfectly. But the <b>Privilege</b> class performs far worse, with recall of <b>0.33</b> on validation and <b>0.52</b> on test. The problem is not the random forest itself; the problem is the combination of low support and difficult minority-class structure.
</p>
<p>
This is a recurring pattern in security machine learning. The classes that dominate the weighted averages are often not the classes that matter most in practice. Missing a rare privilege-escalation event is usually more consequential than slightly over-alerting on common reconnaissance.
</p>

<h2>Interpretation</h2>
<p>
The strong performance on DoS and Probe is intuitive. These classes are well represented and tend to have consistent statistical signatures. Flooding, scan behavior, and service enumeration often leave highly separable traces in engineered connection features.
</p>
<p>
Privilege and Access events are harder. They may be sparse, heterogeneous, and closer to normal application behavior in feature space. They are also the classes where semantic context would help most, but the model sees only precomputed network statistics.
</p>
<p>
This distinction matters because it tells us what the model is actually good at: <b>macro-pattern intrusion triage</b>, not reliable detection of subtle high-value compromise behaviors.
</p>

<h2>Why Random Forest Is Still a Good Choice</h2>
<p>
Even with these limitations, the model choice remains defensible. A random forest gives excellent baseline performance with little tuning, trains quickly, and provides a robust reference point for future improvements. In research workflow terms, it is the right baseline to beat before moving to more complex methods such as gradient boosting, cost-sensitive ensembles, or neural tabular architectures.
</p>
<p>
Importantly, the notebook saves the trained model to <code>network_anomaly_detection_model.joblib</code>, which makes the exercise more than purely academic. The output is deployable as a scoring component, provided its limitations are respected.
</p>

<h2>Limitations</h2>
<p>
NSL-KDD is an educational dataset, not a faithful representation of modern enterprise traffic. That matters because a model can perform extremely well on this benchmark while generalizing poorly to contemporary encrypted, cloud-mediated, or heavily multiplexed environments.
</p>
<p>
The evaluation also remains feature-centric. There is no adversarial robustness analysis, no concept-drift study, and no calibration assessment. A detector that is accurate but poorly calibrated can still create operational problems if its confidence scores are used for automation.
</p>
<p>
Finally, the minority-class issue should be treated as central, not peripheral. For real deployment, I would prioritize class-specific recall improvements over pushing the already excellent global accuracy a few decimal points higher.
</p>

<h2>Conclusion</h2>
<p>
This experiment shows why structured-feature intrusion detection remains a strong domain for classical ensemble learning. The choice of a random forest is justified by the nature of the data, the grouped label design is justified by operational response logic, and the results are strong enough to establish a serious baseline. The important lesson, however, is not that the model achieves 99.5% accuracy. It is that high aggregate metrics can conceal the exact failure modes defenders care about most.
</p>
