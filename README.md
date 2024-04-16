#  Vortel: Deploy Kubernetes MLOps using Python Flask Rest API Client

Vortel is a Python Flask Rest prediction API client used for deploying and scaling ml services ontop of kubernetes.

Vortel defines a simple REST API for an imaginary Machine Learning(ML) model that can be used for testing Docker and Kubernetes.

---

## Why Vortel?
 Vortel is designed to run MLOps on a parallel distributed system, optimized for scalability and DevOps workflows using the API client.
 Vortel is Cloud native and DevOps friendly. Via its Kubernetes-native workflow, specifically the [VortelDeployment CRD](https://docs Vortel.io/en/latest/concepts/VortelDeployment_crd.html) (Custom Resource Definition), DevOps teams can easily fit MLOPS powered services into their existing workflow.

## Containerising a Simple ML Model Scoring Service using Flask and Docker

We start by demonstrating how to achieve this basic competence using the simple Python ML model scoring REST API contained in the `api.py` module, together with the `Dockerfile`, both within the `Vortel` directory, whose core contents are as follows,

```bash
Vortel/
 | Dockerfile
 | Pipfile
 | Pipfile.lock
 | api.py
```

### Defining the Flask Service in the `api.py` Module

This is a Python module that uses the [Flask](http://flask.pocoo.org) framework for defining a web service (`app`), with a function (`prediction`), that executes in response to a HTTP request to a specific URL (or 'route'), thanks to being wrapped by the `app.route` function. For reference, the relevant code is reproduced below,

```python
from flask import Flask, jsonify, make_response, request

app = Flask(__name__)


@app.route('/prediction', methods=['POST'])
def prediction():
    features = request.json['X']
    return make_response(jsonify({'prediction': features}))


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

If running locally - e.g. by starting the web service using `python run api.py` - we would be able reach our function (or 'endpoint') at `http://localhost:5000/prediction`. This function takes data sent to it as JSON (that has been automatically de-serialised as a Python dict made available as the `request` variable in our function definition), and returns a response (automatically serialised as JSON).

In our example function, we expect an array of features, `X`, that we pass to a ML model, which in our example returns those same features back to the caller - i.e. our chosen ML model is the identity function

## Getting Started

- 📖 [Documentation](https://docs Vortel.io/) - Overview of the Vortel docs and related resources
- ⚙️ [Installation](https://docs Vortel.io/en/latest/installation/index.html) - Hands-on instruction on how to install Vortel for production use


## Quick Tour

Let's try out Vortel locally in a minikube cluster!

### ⚙️ Prerequisites:
  * Install latest minikube: https://minikube.sigs.k8s.io/docs/start/
  * Install latest Helm: https://helm.sh/docs/intro/install/
  * Start a minikube Kubernetes cluster: `minikube start --cpus 4 --memory 4096`, if you are using macOS, you should use [hyperkit](https://minikube.sigs.k8s.io/docs/drivers/hyperkit/) driver to prevent the macOS docker desktop [network limitation](https://docs.docker.com/desktop/networking/#i-cannot-ping-my-containers)
  * Check that minikube cluster status is "running": `minikube status`
  * Make sure your `kubectl` is configured with `minikube` context: `kubectl config current-context`
  * Enable ingress controller: `minikube addons enable ingress`

### Defining the Docker Image with the `Dockerfile`

 A `Dockerfile` is essentially the configuration file used by Docker, that allows you to define the contents and configure the operation of a Docker container, when operational. This static data, when not executed as a container, is referred to as the 'image'. For reference, the `Dockerfile` is reproduced below,

```docker
FROM python:3.6-slim
WORKDIR /usr/src/app
COPY . .
RUN pip install pipenv
RUN pipenv install
EXPOSE 5000
CMD ["pipenv", "run", "python", "api.py"]


```

#### Testing

To test that the image can be used to create a Docker container that functions as we expect it to use,

```bash
docker run --rm --name test-api -p 5000:5000 -d danielpickens/Vortel
```

Where we have mapped port 5000 from the Docker container - i.e. the port our ML model scoring service is listening to - to port 5000 on our host machine (localhost). Then check that the container is listed as running using,

```bash
docker ps
```

And then test the exposed API endpoint using,

```bash
curl http://localhost:5000/prediction \
    --request POST \
    --header "Content-Type: application/json" \
    --data '{"X": [1, 2]}'
```

Where you should expect a response along the lines of,

```json
{"prediction":[1,2]}
```

All our test model does is return the input data - i.e. it is the identity function. Only a few lines of additional code are required to modify this service to load a a new ML model from disk and pass new data to it's 'predict' method for generating predictions - 

```bash
docker stop vortel-api
```

#### Pushing the Image to the DockerHub Registry

In order for a remote Docker host or Kubernetes cluster to have access to the image we've created, we need to publish it to an image registry. All cloud computing providers that offer managed Docker-based services will provide private image registries, but we will use the public image registry at DockerHub, for convenience. To push our new image to DockerHub (where my account ID is 'alexioannides') use,

```bash
docker push danielpickens/vortel
```

### 🚧 Install Vortel

Install Vortel:

```bash
git clone https://www.github.com/danielpickens/vortel

cd vortel

python api.py
```

This script will install Vortel along with its dependencies (PostgreSQL and MinIO) on your minikube cluster. 

Note that this installation script is made for development and testing use only.
For production deployment, check out the [Installation Guide](https://docs Vortel.io/en/latest/installation/index.html).

To access Vortel web UI, run the following command and keep the terminal open:

```bash
kubectl --namespace Vortel-system port-forward svc Vortel 8080:80
```

In a separate terminal, run:

```bash Vortel_INITIALIZATION_TOKEN=$(kubectl get secret Vortel-env --namespace Vortel-system -o jsonpath="{.data Vortel_INITIALIZATION_TOKEN}" | base64 --decode)
echo "Open in browser: http://127.0.0.1:8080/setup?token= Vortel_INITIALIZATION_TOKEN"
``` 

Open the URL printed above from your browser to finish admin account setup.

### Dockerfile Configuration

- start by using a pre-configured Docker image (`python:3.6-slim`) that has a version of the [Alpine Linux](https://www.alpinelinux.org/community/) distribution with Python already installed;
 - then copy the contents of the `Vortel` local directory to a directory on the image called `/usr/src/app`;
 - then use `pip` to install the [Pipenv](https://pipenv.readthedocs.io/en/latest/) package for Python dependency management (see the appendix at the bottom for more information on how we use Pipenv);
 - then use Pipenv to install the dependencies described in `Pipfile.lock` into a virtual environment on the image;
 - configure port 5000 to be exposed to the 'outside world' on the running container; and finally,
 - to start our Flask RESTful web service - `api.py`. Note, that here we are relying on Flask's internal [WSGI](https://en.wikipedia.org/wiki/Web_Server_Gateway_Interface) server, whereas in a production setting we would recommend on configuring a more robust option (e.g. Gunicorn), [as discussed here](https://pythonspeed.com/articles/gunicorn-in-docker/).

 Building this custom image and asking the Docker daemon to run it (remember that a running image is a 'container'), will expose our RESTful ML model prediction service on port 5000 as if it were running on a dedicated virtual machine. Refer to the official [Docker documentation](https://docs.docker.com/get-started/) for a more comprehensive discussion of these core concepts.

### Building a Docker Image for the ML Scoring Service

We assume that [Docker is running locally](https://www.docker.com) (both Docker client and daemon), that the client is logged into an account on [DockerHub](https://hub.docker.com) and that there is a terminal open in the this project's root directory. To build the image described in the `Dockerfile` run,

```bash
docker build --tag

### 🍱 Push MLOPs(Apptainer,VertexAI,AWS) to Vortel

First, get an API token and login to the MLOPS CLI:

* Keep the `kubectl port-forward` command in the step above running
* Go to Vortel's API tokens page: http://127.0.0.1:8080/api_tokens
* Create a new API token from the UI, making sure to assign "API" access under "Scopes"
* Copy the login command upon token creation and run as a shell command, e.g.:

    ```bash
    apptainer/vertex Vortel login --api-token {YOUR_TOKEN} --endpoint http://127.0.0.1:8080
    ```

If you don't already have a MLOPS built, run the following commands from the [MLOPS Quickstart Project](https://github.com/danielpickens/Vortel/tree/main/examples/quickstart) to build a sample MLOPS:

```bash
git clone https://github.com/danielpickens/Vortel.git && cd ./examples/quickstart
pip install -r ./requirements.txt
python api.py

```

Push your newly built ML Model to Vortel:

```bash
MLOPS/apptainer/slurm push iris_classifier:latest
```


### 🔧 Install Vortel-image-builder component
 Vortel's image builder feature comes as a separate component, you can install it via the following
script:

```bash
bash <(curl -s "https://raw.githubusercontent.com/DanielPickens/Vortel/MLOPS Vortel-image-builder/main/scripts/quick-install Vortel-image-builder.sh")
```

This will install the `MLOPSRequest` CRD([Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)) and `MLOPS` CRD
in your cluster. Similarly, this script is made for development and testing purposes only.

### 🔧 Install Vortel-deployment component
 Vortel's Deployment feature comes as a separate component, you can install it via the following
script:

```bash
bash <(curl -s "https://raw.githubusercontent.com/DanielPickens/Vortel/MLOPS Vortel-deployment/main/scripts/quick-install Vortel-deployment.sh")
```

This will install the `VortelDeployment` CRD([Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/))
in your cluster and enable the deployment UI on Vortel. Similarly, this script is made for development and testing purposes only.

### 🚢 Deploy MLOPS!

Once the  Vortel-deployment` component was installed, using MLOPS for ML Ops can be pushed to Vortel then can be deployed to your 
Kubernetes cluster and exposed via a Service endpoint. 

A MLOPS Deployment can be created via applying a VortelDeployment resource:

Define your MLOPS deployment in a `my_deployment.yaml` file:

```yaml
apiVersion: resources Vortel.ai/v1alpha1
kind: MLOPSRequest
metadata:
    name: iris-classifier
    namespace: Vortel
spec:
    MLOPSTag: iris_classifier:3oevmqfvnkvwvuqj  # check the tag by `MLOPS list iris_classifier`
---
apiVersion: serving Vortel.ai/v2alpha1
kind: VortelDeployment
metadata:
    name: my-MLOPS-deployment
    namespace: Vortel
spec:
    MLOPS: iris-classifier
    ingress:
        enabled: true
    resources:
        limits:
            cpu: "500m"
            memory: "512Mi"
        requests:
            cpu: "250m"
            memory: "128Mi"
    autoscaling:
        maxReplicas: 10
        minReplicas: 2
    runners:
        - name: iris_clf
          resources:
              limits:
                  cpu: "1000m"
                  memory: "1Gi"
              requests:
                  cpu: "500m"
                  memory: "512Mi"
          autoscaling:
              maxReplicas: 4
              minReplicas: 1
```

Apply the deployment to your minikube cluster:
```bash
kubectl apply -f my_deployment.yaml
```

Now you can check the deployment status via `kubectl get VortelDeployment -n my-MLOPS-deployment`



## Community

-   To report a bug or suggest a feature request, use [GitHub Issues](https://github.com/danielpickens/Vortel/issues/new/choose).



## Contributing

There are many ways to contribute to the project:

-   Report issues you're facing and "Thumbs up" on issues and feature requests that are relevant to you.
-   Investigate bugs and review other developers' pull requests.
-   Contributing code or documentation to the project by submitting a GitHub pull request. 

