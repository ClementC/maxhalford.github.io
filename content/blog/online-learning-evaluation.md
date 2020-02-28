+++
date = "2019-07-13"
draft = true
title = "Methods for evaluating online machine learning models"
+++

## Motivation

Most supervised machine learning algorithms work in the batch setting, whereby they are fitted on a training set offline and predict the outcome of new observations online. The only way for batch machine learning algorithms to learn from new observations is to train them from scratch with both the old and new observations. Meanwhile, some learning algorithms are online, and can predict as well as update themselves when new observations are available. This encompasses any model trained with [stochastic gradient descent](https://leon.bottou.org/publications/pdf/compstat-2010.pdf) -- which includes deep neural networks, [factorisation machines](https://www.csie.ntu.edu.tw/~b97053/paper/Rendle2010FM.pdf), and [SVMs](https://www.cs.huji.ac.il/~shais/papers/ShalevSiSrCo10.pdf) -- as well as [decision trees](https://homes.cs.washington.edu/~pedrod/papers/kdd00.pdf), [metric learning](https://ai.stanford.edu/~ang/papers/icml04-onlinemetric.pdf), and [naïve Bayes](https://people.csail.mit.edu/jrennie/papers/icml03-nb.pdf).

When trained on the same amount of data, online models are usually weaker than batch models. Researchers try to build online models that are guaranteed to reach the same performance as a batch model when the size of the data grows. However, comparing online models to batch models isn't really fair, because they're not meant to solve different problems. Batch models are meant to be used when you can afford to retrain your model from scratch every so often. Online models, on the contrary, are meant to be used when to you want your model to learn from a stream of data, and therefore never have to restart from scratch. Learning from a stream of data is something a batch model can't do, and is very much different than the usual train/test split paradigm that machine learning practitioners are used to. In fact, there are other ways to evaluate the performance of an online model that make more sense than, say, cross-validation. Before getting into this topic, let's talk a bit how to compare an online model to a batch model (which, as I said, doesn't really make sense, but nonetheless helps to develop an understanding).

## Cross-validation

To begin with, I'm going to compare scikit-learn's [`SGDClassifier`](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.SGDClassifier.html) and [`LogisticRegression`](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.LogisticRegression.html#sklearn.linear_model.LogisticRegression). In a nutshell, `SGDClassifier` has the same model parameters as a `LogisticRegression`, but is trained via stochastic gradient descent, and can thus learn from a stream of data -- via the [`partial_fit`](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.SGDClassifier.html#sklearn.linear_model.SGDClassifier.partial_fit) method. Note that scikit-learn provides [a list](https://scikit-learn.org/stable/modules/computing.html#incremental-learning) of it's estimators that support online learning -- they use call this "incremental learning", which is exactly the same thing.

## Progressive validation

In the case of online machine learning, we have another validation tool at our disposal called [progressive validation](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.153.3925&rep=rep1&type=pdf). In an online setting, observations arrive from a stream in sequential order. Each observation can be denoted as $(x_t, y_t)$, where $x_t$ is a set of features, $y_t$ is a label, and $t$ is used to denote time (i.e., it can be an integer or a timestamp). Before updating the model with the pair $(x_t, y_t)$, we can ask the model to predict the output of $x_t$ and thus obtain $\hat{y}_t$. We can then update a live metric by providing it with $y_t$ and $\hat{y}_t$ (common metrics such as accuracy, MSE, and ROC AUC are all sums and can thus be updated online). By doing so, the model is trained with all the data in a single pass, and all the data is as well used as a validation set. Think about that, because it's quite a powerful idea. Moreover, the data is processed in order in which it arrives, which means that it is virtually impossible to introduce [data leakage](https://www.quora.com/Whats-data-leakage-in-data-science) -- including [target leakage](https://www.datarobot.com/wiki/target-leakage/).

In an online setting, progressive validation is a natural method and is often used in practice. For instance, it is mentioned in subsection 5.1 of [*Ad Click Prediction: a View from the Trenches*](https://static.googleusercontent.com/media/research.google.com/fr//pubs/archive/41159.pdf) from researchers at Google for evaluating an ad click–through rate (CTR) prediction model. The authors remark that models based on the gradient of a loss function require computing a prediction anyway, in which case progressive validation can essentially be performed for free. Progressive validation is appealing because it attempts to simulate a live environment wherein the model has to predict the outcome of $x_t$ before the ground truth $y_t$ is made available. For instance in the case of a CTR task, the label $y_t \in \{0, 1\}$ is available once the user has clicked on the ad (i.e., $y_t = 1$), has navigated to another page (i.e., $y_t = 0$), or a given amount of time has passed (i.e., $y_t = 0$). Indeed, in a live environment, there is a delay between the query (i.e., predicting the outcome of $x_t$) and the answer (i.e., when $y_t$ is revealed to the model). Note that machine learning is used in the first place because we want to guess $y_t$ before it happens. The larger the delay, the lesser the chance that $x_t$ and $y_t$ will arrive in perfect sequence. Before $y_t$ is made available, any of $x_{t+1}, x_{t+2}, \dots$ could potentially arrive and require predictions to be made. However, when using progressive validation, we implicitly assume that there is no delay. In other words the model has access to $y_t$ immediately after having produced $\hat{y}_t$. Therefore, progressive validation is not necessarily a faithful simulation of a live environment.

## Delayed progressive validation

In a CTR task, the delay between the query $x_t$ and the answer $y_t$ is usually quite small and can be measured in seconds. However, for other tasks the gap can be quite large because of the nature of the problem. If a model predicts the number of bikes in a shared bike station in 30 minutes, then obviously it will only be able to update itself with the true number of bikes once the 30 minutes have gone by. However, when using progressive validation, the model is given access to the true number of bikes in 30 minutes once it has made a prediction. If the model is then asked to predict the number of bikes in 40 minutes, then it will be cheating because it knows how many bikes are available in 30 minutes. In a live environment this situation can't occur because the future is obviously unknown. However, in a local environment this kind of leakage can occur if one is not careful. To accurately simulate a live environment and thus get a reliable estimate of the performance of a model, we thus need to take into account the delay in arrival times between $x_t$ and $y_t$. The problem with PCV is that it doesn't take said delay into account.

The gold standard is to have a log file with the arrival times of each set of features $x_t$ and each outcome $y_t$. We can then ask the model to make a prediction when $x_t$ arrives, and update itself with $(x_t, y_t)$ once $y_t$ is available. In a fraud detection system for credit card transactions, $x_t$ would contain details about the transaction, whilst $y_t$ would be made available once a human expert has confirmed the transaction as fraudulent or not. However, a log file might not always be available. Indeed, most of the time datasets do not indicate the times at which both the features and the targets arrive. A method for alleviating this issue is called "delayed progressive validation". I've added quotes because I actually coined it myself. The short story is that I wanted to publish a paper on the topic; then I exchanged with [Albert Bifet](http://albertbifet.com/) and he told me that his team had already [published something](https://link.springer.com/article/10.1007/s10618-019-00654-y). I decided to write a blog post instead!

Delayed progressive validation is quite intuitive. Instead of updating the model immediately after it has made a prediction, the idea is to pick a delay $d > 0$ and append the quadruplet $(x_t, y_t, \hat{y}_t, t + d)$ into a sorted list which we'll call $Q$. Once we reach $t + d$, the model is given access to $y_t$ and can therefore be updated, whilst the metric can be updated with $y_t$ and $\hat{y}_t$. We can check if we've reached $t + d$ every time a new observation comes in. The nice thing is that $d$ can be anything you like, and doesn't necessarily have to be the same for every observation. This provides the flexibility of either using a constant, a random variable, or a value that depends on one or more attributes of $x_t$. For instance, in a credit card fraud detection task, it might be that the delay varies according to the credit card issuer. Initially, $Q$ is empty, and grows every time the model makes a prediction. Once the next observation arrives, we loop over $Q$ in insertion order. For each quadruplet in $Q$ which is old enough, we update the model and the metric, before removing the quadruplet from $Q$. Because $Q$ is ordered, we can break the loop over $Q$ whenever a quadruplet is not old enough. Once we have depleted the stream of data, $Q$ will still contain some quadruplets, and so the final step of the procedure is to update the metric with the remaining quadruplets.

On the one hand, delayed progressive validation will perform as many predictions and model updates as progressive validation. The only added cost comes from inserting $(x_t, y_t, \hat{y}_t, t + d)$ into $Q$ so that $Q$ remains sorted. This can be done in $\mathcal{O}(|Q|)$ time by using the bisection method, with $|Q|$ being the length of $Q$. In the special case where the delay $d$ is constant, the bisection method can be avoided because each quadruplet $(x_t, y_t, \hat{y}_t, t + d)$ can simply be inserted at the beginning of $Q$. Other operations, namely comparing timestamps and picking a delay, are trivial. On the other hand, the space complexity is higher than progressive validation because $Q$ has to be maintained in memory. Naturally, the size of the queue is proportional to the delay. For most cases this shouldn't be an issue because the observations are being processed one at a time, which means that quadruplets are added and dropped from the queue at a very similar rate. In practice, if the amount of available memory runs out, then $Q$ can be written to the disk, but this is very much an edge case. Finally, note that progressive validation can be seen as a special case of delayed progressive validation when the delay is set to 0. Indeed, in this case $Q$ will contain at most one element, whilst the predictions and model updates will be perfectly interleaved.