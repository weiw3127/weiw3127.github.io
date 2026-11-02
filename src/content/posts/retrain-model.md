---
title: "Is It Time to Retrain My Model?"
description: "A practical guide to diagnosing data and concept drift in production ML systems, and choosing when to monitor, recalibrate, adjust thresholds, or retrain."
pubDatetime: 2026-11-02T00:00:00+11:00
featured: false
draft: false
tags:
  - machine-learning
canonicalURL: "https://weiwstack.substack.com/p/is-it-time-to-retrain-my-model"
---
One day, I spotted something strange on my dashboard: a swarm of suspicious, nearly identical domains, all seemingly imitating the same big brand. I can’t share the real names, but imagine seeing net-flix.com, net.flix.com, and a lot more like them.

My instincts kicked in right away. Usually, big brands like this would send emails through a small, stable set of domains. Suddenly having so many aliases was pretty unusual.

This was the late 2010s, the boom years of the subscription economy, and scammers had started impersonating big brands to trick their customers. And the worst part is, our detector let many of these through, because their content and structure looked very similar to the legitimate emails they were copying.

This was my first-hand experience with data drift; the input data isn’t covered by our training set, and our model lacks the ability to detect these cases. And data drift is just one of many causes of production failures.

# Model Failure

One of the common issues that cause ML failure in production is system failure, such as data pipeline issues, network outages, dependency failures, etc. These are usually easy to spot. If you see a sudden drop on the dashboard, chances are something is broken in the system. A software problem requires a software solution, which is not what I’m going to cover in this post.  What’s more related to my ML experience is drift. This post focuses on two forms: data drift and concept drift. For a broader overview, see [Chip Huyen’s discussion of data distribution shifts](https://huyenchip.com/2022/02/07/data-distribution-shifts-and-monitoring.html#detecting-data-shifts).

## Data Drift

Data Drift, also known as **covariate shift**, means the input feature distribution has moved away from what our training distribution covered.

<p align="center">
  P<sub>prod</sub>(x) ≠ P<sub>train</sub>(x)
</p>

The relationship between input and label, P(y ∣ x), is still the same, but our model is now handling features from regions it’s not familiar with, so its predictions become less reliable

I ran into this during the pandemic. When lockdowns began, lots of people were forced to start buying groceries online. So our platform suddenly attracted a whole new demographic, very different from our usual customer base.

To diagnose this data drift, an obvious starting point is observing the distribution summaries: mean, percentiles, missing rate, and outlier rate. These kinds of descriptive statistics help us see whether the data has a broad location drift or is spreading out.

A more rigorous approach is statistical drift detection, which comes in two kinds: **Univariate detection** checks every feature individually. **Multivariate detection** checks how features behave together.

### Univariate Data Drift: Checking One Feature at a Time

**Univariate detection** examines drift for each feature on its own, which is usually a great starting point. Has the production distribution of order values changed? Are we currently having more users visiting from mobile? By knowing the change in a single feature, we can narrow down which parts of our input data have shifted and know where to dig next.

We can compare production data with the training set, or with an earlier period when the model was still performing well.

#### Kolmogorov-Smirnov (KS) Test

The [two-sample Kolmogorov-Smirnov (KS) test](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.ks_2samp.html) checks whether two groups of samples are drawn from the same continuous distribution.  KS test is a nonparametric test. It works by comparing the cumulative distribution of our production and training features and maximises the absolute distance between them.

<p align="center">
  KS = max<sub>x</sub> |F<sub>train</sub>(x) − F<sub>prod</sub>(x)|
</p>

This comparison can reveal changes that a single summary, like the average, would hide. For example, if we just see the average order value, it might look fairly stable. But if we look into the sample distribution, we can see the small and large orders have both become more common.

But one thing to be careful about. With more observations, even a tiny difference can become statistically significant. For a production traffic of billions of transactions, a small p-value alone doesn’t tell us whether the change matters.

#### Chi-squared Test

For categorical features, we can use Chi-squared test. Suppose our training data is mostly generated from desktop traffic, but mobile users are now growing in proportion. Chi-squared test can let us examine whether the device mix has changed beyond what sampling variation would explain.

Be careful with categories that have very small expected counts, because the usual chi-squared approximation becomes unreliable (see [SciPy’s `chi2_contingency` documentation](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.chi2_contingency.html)).

If we notice a drift in a feature, we then need to check whether the drifting region matters to the model. Suppose the feature drifts significantly, but the model barely uses that feature. The operational impact may be negligible. Conversely, a small shift in a critical feature near a decision threshold may strongly affect predictions. We can check feature importance and SHAP values.

### Multivariate Drift Detection: Checking Features Together

Looking at this example:

| Dataset | Half the rows | Other half |
| --- | --- | --- |
| Training | x<sub>1</sub> = 0, x<sub>2</sub> = 0 | x<sub>1</sub> = 1, x<sub>2</sub> = 1 |
| Production | x<sub>1</sub> = 0, x<sub>2</sub> = 1 | x<sub>1</sub> = 1, x<sub>2</sub> = 0 |

If we look at only one feature at a time, we find that each feature is evenly split into 1 and 0. But put them together, we can see that what’s actually changing is how these two features are related to each other. Univariate checks would miss this change in their joint distribution. This is where multivariate tests come in handy.

#### Domain Classifier

We can consider this as a classification problem. If our classifier can easily distinguish the training and production sets, chances are there’s a significant difference between them.

A simple classifier can be trained with:

1. Label the training data 0 and the production data 1
2. Measure the AUC on a held-out set

If the AUC stays close to 0.5, that indicates the two datasets are difficult to distinguish, and there’s little evidence of drift. But if, on the other hand, the AUC is high, there is meaningful multivariate distribution drift.

One caveat: the drift is usually more subtle than the normal classification case. A simple model might miss the shift entirely ([Rabanser et al., 2019, Section 3](https://arxiv.org/html/1810.11953v4#S3)). A practical recommendation is to use a flexible one, such as a tree-based model.

#### Maximum Mean Discrepancy (MMD)

[Maximum Mean Discrepancy (MMD)](https://docs.seldon.ai/alibi-detect/cd/methods/offline/mmddrift) is another method for detecting multivariate drift. Generally speaking, MMD uses kernels to calculate the similarity between two data points; then it compares whether points within a dataset look more alike than the points across the two datasets.

## Concept Drift

Concept drift means the relationship between inputs and outcomes has changed, which means P(y ∣ x) is no longer the same; even with the same inputs, the meaning changes.

<p align="center">
  P<sub>prod</sub>(y ∣ x) ≠ P<sub>train</sub>(y ∣ x)
</p>

When bad people change phishing tactics, the old patterns we captured no longer indicate fraud. Customers may act differently due to our promotion campaigns. The investment volume changed because the official launched a new trading policy. Here are some examples I’ve encountered in my career. In each case, the inputs might still look familiar, but what they would lead to has changed.

### Evaluate Production Data to Find Blind Spots

A practical starting point is to collect fresh samples from production and test on our model, evaluating on the training metric and seeing whether the performance drops.

However, the overall score can only tell us part of the story. Concept drift can sometimes be a local problem. An overall metric might look fair while the performance might already deteriorate for a particular subgroup.

We can locate the local drift by breaking down performance by customer segment, geography, device type, or combinations of features. With each group’s error rate and sample count, we can evaluate error coverage: how much of the model’s total errors come from which group.

If the drift is a local problem, understanding this pattern can lead us to a better active learning strategy.

### Train a Recent-Data Challenger

We can also just train a new challenger using new data! The comparison between the new and old models would show us direct evidence of drift.

Let’s say we can do things like:

1. The existing model, evaluated on the held-out week.
2. The challenger trained on recent data, evaluated on the same held-out week.

If the challenger performs significantly better, the next step is to find where the improvement comes from.

Were these cases always difficult for the existing model, or have previous patterns stopped working? If our model always struggled with those cases and they have become more common, the root problem might be more like training coverage. However, if similar inputs now lead to different outcomes, we can almost certainly call it concept drift.

# Choosing a Response

Discovering data or concept drift doesn’t necessarily mean the model is failing. Our action depends on observing what has changed and how much it matters. Once we have a full picture of the problem, we can consider one of several responses:

## Keep Monitoring

When the input distribution P(X) changes, but model performance and business outcomes are still stable. It’s possible the shifted features have less impact on the overall situation.

For example, we have more production traffic coming from mobile devices. Since our model generalises the signal pretty well, it can handle these unseen scenarios. If the precision, recall, calibration, and other business metrics remain acceptable, we may not need to change anything; we can just keep monitoring.

## Recalibrate Probabilities

In some cases, a classifier may still be able to separate positive and negative cases while its predicted probability has become unreliable.

Suppose cases scored 0.8 used to be positive 80% of the time, but now it’s only about 50%. The scores our classifier produced can still prioritise cases, but the probability interpretation has changed. In this case, we need to recalibrate.

ROC-AUC can show us the discrimination capability of our classifier. To investigate calibration, we can compare the predicted probabilities with observed probabilities across score buckets.

Instead of retraining the model, we can use [calibration methods](https://scikit-learn.org/stable/modules/calibration.html) such as temperature scaling or sigmoid calibration, or an intercept adjustment to make the model’s scores more reflective of reality. This calibration should be fit using recent production data and kept separate from the classifier’s training data.

## Adjust the Operation Threshold

Sometimes the model remains useful, but the decision we want to make from its predictions has changed.

Our ML system relied on human review in many situations. Suppose our model assigns each case a score between 0 and 1. We send cases the model is uncertain about to expert reviewers, which are those with scores in a middle range, let’s say 0.45 to 0.8. If our daily review capacity drops from 10,000 to 5,000 cases, we need to adjust these thresholds to prioritise cases where a false negative or false positive would have the most serious consequences, while keeping the workload within our team’s capacity.

Maybe our human-in-the-loop reviewing capacity drops from 10,000 to 5,000 cases a day. Or the cost of false positives increases. Changes in class prevalence can also affect the value of our current decisions.

Those are the case that business back ground change affect how the prediction will be used. In these situations, we can evaluate a different operating threshold against the change on the business side, including updated costs, capacity, and objectives. With the adjustment in threshold, the precision, recall, review volume, or what else changes, the business impact can be reflected.

Changing the threshold changes which cases receive an action. The underlying prediction scores stay the same.

## Retrain or Revise the Model

Retraining the model is a big process, but sometimes it’s inevitable if our model is unable to handle the recent data. I’d retrain when the preprocessing pipeline has changed, when P(Y ∣ X) has changed significantly, or when new target data shows up.

We also need to decide a training data window:

- Recent-only window: Use when old relationships are actively misleading; less data and higher variance
- Expanding window: Use when the old patterns still hold, and the new data adds coverage. We keep all historical data and add the recent data on top.
- Recency weighting: Use when old data still helps but recent observations should matter more. We keep everything but give newer samples higher weight.
