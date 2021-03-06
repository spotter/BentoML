Deploying to Kubernetes Cluster
===============================

This guide demostrates how to deploy a model server built with BentoML to Kubernetes cluster.

Kubernetes is an open-source system for automating deployment, scaling, and management of
containerized applications. It is the de-facto solution for deploying applications today.
Machine larning services also can take advantage of Kubernetes' ability to quickly deploy
and scale base on demand.


Prerequsities
-------------

Before starting this guide, make sure you have the following:

* A kubernetes installed cluster. `minikube` is used in this guide.

    * Kubernetes guide: https://kubernetes.io/docs/setup/

    * `minikube` install instruction: https://kubernetes.io/docs/tasks/tools/install-minikube/

* `kubectl` CLI tool

    * install instruction: https://kubernetes.io/docs/tasks/tools/install-kubectl/

* Docker and Docker Hus is properly configured on your system

    * Install instruction: https://docs.docker.com/install

* Python 3.6 or above and required packages: `bentoml` and `scikit-learn`

    * .. code-block:: bash

            pip install bentoml scikit-learn



Kubernetes deployment with BentoML
----------------------------------

This guide builds a BentoService with iris classifier model, and deploys the
BentoService to Kubernetes as API model server.


Use the IrisClassifier BentoService from the quick start guide.

.. code-block:: bash

    git clone git@github.com:bentoml/BentoML.git
    python ./bentoml/guides/quick-start/main.py


Use BentoML CLI tool to get the information of IrisClassifier created above

.. code-block:: bash

    bentoml get IrisClassifier:latest

    # Sample output
    {
      "name": "IrisClassifier",
      "version": "20200121141808_FE78B5",
      "uri": {
        "type": "LOCAL",
        "uri": "/Users/bozhaoyu/bentoml/repository/IrisClassifier/20200121141808_FE78B5"
      },
      "bentoServiceMetadata": {
        "name": "IrisClassifier",
        "version": "20200121141808_FE78B5",
        "createdAt": "2020-01-21T22:18:25.079723Z",
        "env": {
          "condaEnv": "name: bentoml-IrisClassifier\nchannels:\n- defaults\ndependencies:\n- python=3.7.3\n- pip\n",
          "pipDependencies": "bentoml==0.5.8\nscikit-learn",
          "pythonVersion": "3.7.3"
        },
        "artifacts": [
          {
            "name": "model",
            "artifactType": "SklearnModelArtifact"
          }
        ],
        "apis": [
          {
            "name": "predict",
            "handlerType": "DataframeHandler",
            "docs": "BentoService API"
          }
        ]
      }
    }

=================================
Deploy BentoService to Kubernetes
=================================

BentoML provides a convenient way of containerizing the model API server with Docker. To
create a docker container image for the sample model above:

1. Find the file directory of the SavedBundle with bentoml get command, which is directory
structured as a docker build context.

2. Running docker build with this directory produces a docker image containing the model
API server

.. code-block:: bash

    saved_path=$(bentoml get IrisClassifier:latest -q | jq -r ".uri.uri")

    # Replace {docker_username} with your Docker Hub username
    docker build -t {docker_username}/iris-classifier $saved_path
    docker push {docker_username}/iris-classifier


The following is an example YAML file for specifying the resources required to run and
expose a BentoML model server in a Kubernetes cluster. Replace {docker_username} with
your Docker Hub username and save it to iris-classifier.yaml

.. code-block:: yaml

    #iris-classifier.yaml

    apiVersion: v1
    kind: Service
    metadata:
        labels:
            app: iris-classifier
        name: iris-classifier
    spec:
        ports:
        - name: predict
          port: 5000
          targetPort: 5000
        selector:
          app: iris-classifier
        type: LoadBalancer
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
        labels:
            app: iris-classifier
        name: iris-classifier
    spec:
        selector:
            matchLabels:
                app: iris-classifier
        template:
            metadata:
                labels:
                    app: iris-classifier
            spec:
                containers:
                - image: {docker_username}/iris-classifier
                  imagePullPolicy: IfNotPresent
                  name: iris-classifier
                  ports:
                  - containerPort: 5000


Use `kubectl` CLI to deploy model server to Kubernetes cluster.

.. code-block:: bash

    kubectl apply -f iris-classifier.yaml


Make prediction with `curl`:

.. code-block:: bash

    curl -i \
    --request POST \
    --header "Content-Type: application/json" \
    --data '[[5.1, 3.5, 1.4, 0.2]]' \
    ${minikube ip}:5000/predict


============================================
Monitor model server metrics with Prometheus
============================================

Setup:

Before starting this section, make sure you have the following:

* Prometheus installed on your Kubernetes cluster

  * installation instruction: https://github.com/coreos/kube-prometheus


BentoML API server provides Prometheus support out of the box. It comes with a “/metrics”
endpoint which includes the essential metrics for model serving and the ability to create
and customize new metrics base on needs.

To enable Prometheus monitoring on the deployed model API server, update the YAML file
with Prometheus related annotations. Change the deployment spec as the following, and
replace `{docker_username}` with your Docker Hub username:


.. code-block:: bash

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: pet-classifier
      name: pet-classifier
    spec:
      selector:
        matchLabels:
          app: pet-classifier
      template:
        metadata:
          labels:
            app: pet-classifier
          annotations:
            prometheus.io/scrape: true
            prometheus.io/port: 5000
        spec:
          containers:
          - image: {docker_username}/pet-classifier
            name: pet-classifier
            ports:
            - containerPort: 5000


Apply the changes to enable monitoring:

.. code-block:: bash

    kubectl apply -f iris-classifier.yaml



=================
Remove deployment
=================

.. code-block:: bash

    kubectl delete -f iris-classifier.yaml

