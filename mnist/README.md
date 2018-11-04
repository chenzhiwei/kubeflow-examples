# Training MNIST using Kubeflow, S3, and Argo.

This example guides you through the process of taking an example model, modifying it to run better within Kubeflow, and serving the resulting trained model. We will be using Argo to manage the workflow, Tensorflow's S3 support for saving model training info, Tensorboard to visualize the training, and Kubeflow to deploy the Tensorflow operator and serve the model.

## Prerequisites

Before we get started there a few requirements.

## 1.Kubernetes Cluster Environment

* ICP v3.1.0/v3.1.1
* Kubernetes v1.11.1/v1.11.3

## 2.Setup Kubeflow

### Download ksonnet

```
curl -fksSL https://github.com/ksonnet/ksonnet/releases/download/v0.13.0/ks_0.13.0_linux_amd64.tar.gz | tar --strip-components=1 -xvz -C /usr/local/bin/ ks_0.13.0_linux_amd64/ks
```

### Install kubeflow

```
git clone -b v0.3.2 https://github.com/kubeflow/kubeflow
./kubeflow/scripts/kfctl.sh init kf-app --platform none
cd kf-app
../kubeflow/scripts/kfctl.sh generate k8s
../kubeflow/scripts/kfctl.sh apply k8s
```

## 3.Setup Minio

### Deploy Minio

```
docker run --name=minio --net=host -d -v /var/lib/minio:/data minio/minio:RELEASE.2018-10-25T01-27-03Z server /data
```

### Get Minio key and secret

```
cat /var/lib/minio/.minio.sys/config/config.json  | head

{
	"version": "31",
	"credential": {
		"accessKey": "M99ZBO148UJTJQCR2ZOY",
		"secretKey": "yLb_m23bBmXA8f3gkefP-85HuBKjQ0A9OzdhR8SI",
		"expiration": "1970-01-01T00:00:00Z",
		"status": "enabled"
	},
	"region": "",
	"worm": "off",
```

### Create Minio bucket

Open Minio dashbaord in browser: http://minio-host:9000

Create bucket: `mnist-model`


## Download argo

```
curl -sSL -o /usr/local/bin/argo https://github.com/argoproj/argo/releases/download/v2.2.1/argo-linux-amd64
chmod +x /usr/local/bin/argo
```

## Create tf service account

```
kubectl -n kubeflow apply -f tf-user.yaml
```

## Create secret for workflow

```
export NAMESPACE=kubeflow

export S3_ENDPOINT=9.30.218.76:9000
export AWS_ENDPOINT_URL=http://${S3_ENDPOINT}
export AWS_ACCESS_KEY_ID=M99ZBO148UJTJQCR2ZOY
export AWS_SECRET_ACCESS_KEY=yLb_m23bBmXA8f3gkefP-85HuBKjQ0A9OzdhR8SI
export AWS_REGION=us-west-2
export BUCKET_NAME=mnist-model
export S3_USE_HTTPS=0
export S3_VERIFY_SSL=0

kubectl -n ${NAMESPACE} create secret generic aws-creds --from-literal=awsAccessKeyID=${AWS_ACCESS_KEY_ID} \
 --from-literal=awsSecretAccessKey=${AWS_SECRET_ACCESS_KEY}
```

## Submit training workflow

```
export S3_DATA_URL=s3://${BUCKET_NAME}/data/mnist/
export S3_TRAIN_BASE_URL=s3://${BUCKET_NAME}/models
export JOB_NAME=myjob-$(uuidgen  | cut -c -5 | tr '[:upper:]' '[:lower:]')
export TF_MODEL_IMAGE=siji/mnist-model:v1.0.0
export TF_WORKER=3
export MODEL_TRAIN_STEPS=200

argo submit model-train.yaml -n ${NAMESPACE} --serviceaccount tf-user \
    -p aws-endpoint-url=${AWS_ENDPOINT_URL} \
    -p s3-endpoint=${S3_ENDPOINT} \
    -p aws-region=${AWS_REGION} \
    -p tf-model-image=${TF_MODEL_IMAGE} \
    -p s3-data-url=${S3_DATA_URL} \
    -p s3-train-base-url=${S3_TRAIN_BASE_URL} \
    -p job-name=${JOB_NAME} \
    -p tf-worker=${TF_WORKER} \
    -p model-train-steps=${MODEL_TRAIN_STEPS} \
    -p s3-use-https=${S3_USE_HTTPS} \
    -p s3-verify-ssl=${S3_VERIFY_SSL} \
    -p namespace=${NAMESPACE}
```

## Modifying existing examples

Many examples online use models that are unconfigurable, or don't work well in distributed mode. We will modify one of these [examples](https://github.com/tensorflow/tensorflow/blob/9a24e8acfcd8c9046e1abaac9dbf5e146186f4c2/tensorflow/examples/learn/mnist.py) to be better suited for distributed training and model serving.

### Prepare model

There is a delta between existing distributed mnist examples and what's needed to run well as a TFJob.

Basically, we must:

1. Add options in order to make the model configurable.
1. Use `tf.estimator.train_and_evaluate` to enable model exporting and serving.
1. Define serving signatures for model serving.

The resulting model is [model.py](model.py).

## Preparing your Kubernetes Cluster

With our data and workloads ready, now the cluster must be prepared. We will be deploying the TF Operator, and Argo to help manage our training job.

In the following instructions we will install our required components to a single namespace.  For these instructions we will assume the chosen namespace is `tfworkflow`:

### Deploying Tensorflow Operator and Argo.

We are using the Tensorflow operator to automate the deployment of our distributed model training, and Argo to create the overall training pipeline. The easiest way to install these components on your Kubernetes cluster is by using Kubeflow's ksonnet prototypes.

```
NAMESPACE=tfworkflow
APP_NAME=my-kubeflow
ks init ${APP_NAME}
cd ${APP_NAME}

ks registry add kubeflow github.com/kubeflow/kubeflow/tree/v0.2.4/kubeflow

ks pkg install kubeflow/core@v0.2.4
ks pkg install kubeflow/argo

# Deploy TF Operator and Argo
kubectl create namespace ${NAMESPACE}
ks generate core kubeflow-core --name=kubeflow-core --namespace=${NAMESPACE}
ks generate argo kubeflow-argo --name=kubeflow-argo --namespace=${NAMESPACE}

ks apply default -c kubeflow-core
ks apply default -c kubeflow-argo

# Switch context for the rest of the example
kubectl config set-context $(kubectl config current-context) --namespace=${NAMESPACE}
cd -

# Create a user for our workflow
kubectl apply -f tf-user.yaml
```

You can check to make sure the components have deployed:

```
$ kubectl get pods -l name=tf-job-operator
NAME                              READY     STATUS    RESTARTS   AGE
tf-job-operator-78757955b-2glvj   1/1       Running   0          1m

$ kubectl get pods -l app=workflow-controller
NAME                                   READY     STATUS    RESTARTS   AGE
workflow-controller-7d8f4bc5df-4zltg   1/1       Running   0          1m

$ kubectl get crd
NAME                    AGE
tfjobs.kubeflow.org     1m
workflows.argoproj.io   1m

$ argo list
NAME   STATUS   AGE   DURATION
```

### Creating secrets for our workflow and setting S3 variables.

For fetching and uploading data, our workflow requires S3 credentials and variables. These credentials will be provided as kubernetes secrets, and the variables will be passed into the workflow. Modify the below values to suit your environment.

```
export S3_ENDPOINT=s3.us-west-2.amazonaws.com  #replace with your s3 endpoint in a host:port format, e.g. minio:9000
export AWS_ENDPOINT_URL=https://${S3_ENDPOINT} #use http instead of https for default minio installs
export AWS_ACCESS_KEY_ID=xxxxx
export AWS_SECRET_ACCESS_KEY=xxxxx
export AWS_REGION=us-west-2
export BUCKET_NAME=mybucket
export S3_USE_HTTPS=1 #set to 0 for default minio installs
export S3_VERIFY_SSL=1 #set to 0 for defaul minio installs

kubectl create secret generic aws-creds --from-literal=awsAccessKeyID=${AWS_ACCESS_KEY_ID} \
 --from-literal=awsSecretAccessKey=${AWS_SECRET_ACCESS_KEY}
```

## Defining your training workflow

This is the bulk of the work, let's walk through what is needed:

1. Train the model
1. Export the model
1. Serve the model

Now let's look at how this is represented in our [example workflow](model-train.yaml)

The argo workflow can be daunting, but basically our steps above extrapolate as follows:

1. `get-workflow-info`: Generate and set variables for consumption in the rest of the pipeline.
1. `tensorboard`: Tensorboard is spawned, configured to watch the S3 URL for the training output.
1. `train-model`: A TFJob is spawned taking in variables such as number of workers, what path the datasets are at, which model container image, etc. The model is exported at the end.
1. `serve-model`: Optionally, the model is served.

With our workflow defined, we can now execute it.

## Submitting your training workflow

First we need to set a few variables in our workflow. Make sure to set your docker registry or remove the `IMAGE` parameters in order to use our defaults:

```
DOCKER_BASE_URL=docker.io/elsonrodriguez # Put your docker registry here
export S3_DATA_URL=s3://${BUCKET_NAME}/data/mnist/
export S3_TRAIN_BASE_URL=s3://${BUCKET_NAME}/models
export JOB_NAME=myjob-$(uuidgen  | cut -c -5 | tr '[:upper:]' '[:lower:]')
export TF_MODEL_IMAGE=${DOCKER_BASE_URL}/mytfmodel:1.7
export TF_WORKER=3
export MODEL_TRAIN_STEPS=200
```

Next, submit your workflow.

```
argo submit model-train.yaml -n ${NAMESPACE} --serviceaccount tf-user \
    -p aws-endpoint-url=${AWS_ENDPOINT_URL} \
    -p s3-endpoint=${S3_ENDPOINT} \
    -p aws-region=${AWS_REGION} \
    -p tf-model-image=${TF_MODEL_IMAGE} \
    -p s3-data-url=${S3_DATA_URL} \
    -p s3-train-base-url=${S3_TRAIN_BASE_URL} \
    -p job-name=${JOB_NAME} \
    -p tf-worker=${TF_WORKER} \
    -p model-train-steps=${MODEL_TRAIN_STEPS} \
    -p s3-use-https=${S3_USE_HTTPS} \
    -p s3-verify-ssl=${S3_VERIFY_SSL} \
    -p namespace=${NAMESPACE}
```

Your training workflow should now be executing.

You can verify and keep track of your workflow using the argo commands:
```
$ argo list
NAME                STATUS    AGE   DURATION
tf-workflow-h7hwh   Running   1h    1h

$ argo get tf-workflow-h7hwh
```

## Monitoring

There are various ways to monitor workflow/training job. In addition to using `kubectl` to query for the status of `pods`, some basic dashboards are also available.

### Argo UI

The Argo UI is useful for seeing what stage your worfklow is in:

```
PODNAME=$(kubectl get pod -l app=argo-ui -n${NAMESPACE} -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward ${PODNAME} 8001:8001
```

You should now be able to visit [http://127.0.0.1:8001](http://127.0.0.1:8001) to see the status of your workflows.

### Tensorboard

Tensorboard is deployed just before training starts. To connect:

```
PODNAME=$(kubectl get pod -l app=tensorboard-${JOB_NAME} -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward ${PODNAME} 6006:6006
```

Tensorboard can now be accessed at [http://127.0.0.1:6006](http://127.0.0.1:6006).

## Using Tensorflow serving

### Install client requirements

```
apt install python-pip python-setuptools --no-install-recommends
pip install -r requirements.txt
```

### Submit and query result

By default the workflow deploys our model via Tensorflow Serving. Included in this example is a client that can query your model and provide results:

```
SERVICE_IP=$(kubectl -n kubeflow get service -l app=mnist-${JOB_NAME} -o jsonpath='{.items[0].spec.clusterIP}')
TF_MODEL_SERVER_HOST=$SERVICE_IP TF_MNIST_IMAGE_PATH=data/7.png python mnist_client.py
```

This should result in output similar to this, depending on how well your model was trained:
```
outputs {
  key: "classes"
  value {
    dtype: DT_UINT8
    tensor_shape {
      dim {
        size: 1
      }
    }
    int_val: 7
  }
}
outputs {
  key: "predictions"
  value {
    dtype: DT_FLOAT
    tensor_shape {
      dim {
        size: 1
      }
      dim {
        size: 10
      }
    }
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 0.0
    float_val: 1.0
    float_val: 0.0
    float_val: 0.0
  }
}


............................
............................
............................
............................
............................
............................
............................
..............@@@@@@........
..........@@@@@@@@@@........
........@@@@@@@@@@@@........
........@@@@@@@@.@@@........
........@@@@....@@@@........
................@@@@........
...............@@@@.........
...............@@@@.........
...............@@@..........
..............@@@@..........
..............@@@...........
.............@@@@...........
.............@@@............
............@@@@............
............@@@.............
............@@@.............
...........@@@..............
..........@@@@..............
..........@@@@..............
..........@@................
............................
Your model says the above number is... 7!
```

You can also omit `TF_MNIST_IMAGE_PATH`, and the client will pick a random number from the mnist test data. Run it repeatedly and see how your model fares!

### Disabling Serving

Model serving can be turned off by passing in `-p model-serving=false` to the `model-train.yaml` workflow. Then if you wish to serve your model after training, use the `model-deploy.yaml` workflow. Simply pass in the desired finished argo workflow as an argument:

```
WORKFLOW=<the workflowname>
argo submit model-deploy.yaml -n ${NAMESPACE} -p workflow=${WORKFLOW} --serviceaccount=tf-user
```

## Submitting new argo jobs

If you want to rerun your workflow from scratch, then you will need to provide a new `job-name` to the argo workflow. For example:

```
#We're re-using previous variables except JOB_NAME
export JOB_NAME=myawesomejob

argo submit model-train.yaml -n ${NAMESPACE} --serviceaccount tf-user \
    -p aws-endpoint-url=${AWS_ENDPOINT_URL} \
    -p s3-endpoint=${S3_ENDPOINT} \
    -p aws-region=${AWS_REGION} \
    -p tf-model-image=${TF_MODEL_IMAGE} \
    -p s3-data-url=${S3_DATA_URL} \
    -p s3-train-base-url=${S3_TRAIN_BASE_URL} \
    -p job-name=${JOB_NAME} \
    -p tf-worker=${TF_WORKER} \
    -p model-train-steps=${MODEL_TRAIN_STEPS} \
    -p namespace=${NAMESPACE}
```

## Conclusion and Next Steps

This is an example of what your machine learning pipeline can look like. Feel free to play with the tunables and see if you can increase your model's accuracy (increasing `model-train-steps` can go a long way).
