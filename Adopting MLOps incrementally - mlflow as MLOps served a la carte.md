# Adopting MLOps incrementally - mlflow as MLOps served a la carte

# What is MLOps?

MLOps is a evolution of DevOps.

# Components of mlflow


# Using Databrick's community mlflow

While it is possible (and may be recommended!) to self-host a mlflow server, it is more expedient to just use cloud service that offer a managed/hosted mlflow out of the box. Turns out the community edition of Databrick have this: see the guide [here](https://medium.com/the-point-collections/intro-to-mlflow-with-colab-part-1-2-beb80c960ad9), then sign up for an account and create an instance.


# Create a mlflow project



# Saving mlflow Models

According to the official doc:

> An MLflow Model is a standard format for packaging machine learning models that can be used in a variety of downstream tools—for example, real-time serving through a REST API or batch inference on Apache Spark. The format defines a convention that lets you save a model in different “flavors” that can be understood by different downstream tools.

The most important take-away is that it is a standard format that is designed to unify many different ML/AI models out there: whether it is sklearn, tensorflow/pyTorch, keras, or other similar frameworks/libraries.

To work with them, we need to know the basics: How to save/load a model, and (later on) how to deploy a model to production. Actually there's some nauance in the concept here:

A `save` is specifically for "saving" model data into a local directory only. A `log` on the other hand will submit it to the mlflow server to be centrally managed. Since a `load` can load model locally or from the server, it is not exactly symmetric with respect to `save`.

One possible use case for `load` is online training: suppose a model is deployed in production, and some metric/log/monitoring service is run alongside a user feedback function to collect example data where the model made the wrong prediction. These new data are feed back into the training pipeline for re-training. But we don't want to train from scratch - then it make sense to `load` the current version and start from there.

# Serving mlflow Models

Unfortunately my ideal use case cannot yet be realized using mlflow now: see this github [issue](https://github.com/mlflow/mlflow/issues/4219). Basically mlflow server currently does not support webhook/event notification.

