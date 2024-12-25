# Pipeline deployment a Flask API on AWS

This is the project starter repo for the course Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.



## Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found <a href="https://docs.docker.com/install/" target="_blank">here</a>.
* Git: <a href="https://git-scm.com/downloads" target="_blank">Download and install Git</a> for your system. 
* Code editor: You can <a href="https://code.visualstudio.com/download" target="_blank">download and install VS code</a> here.
* AWS Account
* Python version between 3.7 and 3.9. Check the current version using:
```bash
#  Mac/Linux/Windows 
python --version
```
You can download a specific release version from <a href="https://www.python.org/downloads/" target="_blank">here</a>.

* Python package manager - PIP 19.x or higher. PIP is already installed in Python 3 >=3.4 downloaded from python.org . However, you can upgrade to a specific version, say 20.2.3, using the command:
```bash
#  Mac/Linux/Windows Check the current version
pip --version
# Mac/Linux
pip install --upgrade pip==20.2.3
# Windows
python -m pip install --upgrade pip==20.2.3
```
* Terminal
   * Mac/Linux users can use the default terminal.
   * Windows users can use either the GitBash terminal or WSL. 
* Command line utilities:
  * AWS CLI installed and configured using the `aws configure` command. Another important configuration is the region. Do not use the us-east-1 because the cluster creation may fails mostly in us-east-1. Let's change the default region to:
  ```bash
  aws configure set region us-east-2  
  ```
  Ensure to create all your resources in a single region. 
  * EKSCTL installed in your system. Follow the instructions [available here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html#installing-eksctl) or <a href="https://eksctl.io/introduction/#installation" target="_blank">here</a> to download and install `eksctl` utility. 
  * The KUBECTL installed in your system. Installation instructions for kubectl can be found <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl/" target="_blank">here</a>. 


## Initial setup

1. Fork the <a href="https://github.com/udacity/cd0157-Server-Deployment-and-Containerization" target="_blank">Server and Deployment Containerization Github repo</a> to your Github account.
1. Locally clone your forked version to begin working on the project.
```bash
git clone https://github.com/SudKul/cd0157-Server-Deployment-and-Containerization.git
cd cd0157-Server-Deployment-and-Containerization/
```
1. These are the files relevant for the current project:
```bash
.
├── Dockerfile 
├── README.md
├── aws-auth-patch.yml #ToDo
├── buildspec.yml      #ToDo
├── ci-cd-codepipeline.cfn.yml #ToDo
├── iam-role-policy.json  #ToDo
├── main.py
├── requirements.txt
├── simple_jwt_api.yml
├── test_main.py  #ToDo
└── trust.json     #ToDo 
```

     
## Project Steps
### Running the App Locally
1. Set up the environment
```bash
# A secret text string to be used to creating a JWT
export JWT_SECRET='myjwtsecret' && export LOG_LEVEL=DEBUG
# Verify
echo $JWT_SECRET
echo $LOG_LEVEL
```

2. Run the app
```bash
python main.py
```

3. Verify
Open http://127.0.0.1:8080/ in a new browser OR run 

```bash
curl --request GET http://localhost:8080/
```

Result:
```bash
thuyle@Admin-PC:~$ curl --request GET http://localhost:8080/
"Healthy"
```

- Access endpoint /auth
To try the /auth endpoint
```bash
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
```

```bash
echo $TOKEN
```

Result:
```bash
thuyle@Admin-PC:~$ echo $TOKEN
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE3MzYzNDI1OTQsIm5iZiI6MTczNTEzMjk5NCwiZW1haWwiOiJhYmNAeHl6LmNvbSJ9.CnRArVJA-8E0p8s9ASdk0gTc2BBnL-NrSpOs9WwrMWI
```

- Access endpoint /contents
To try the /contents endpoint
```bash
curl --request GET 'http://localhost:8080/contents' -H "Authorization: Bearer ${TOKEN}" | jq .
```

Result:
```bash
{
  "email": "abc@xyz.com",
  "exp": 1736342594,
  "nbf": 1735132994
}
```

### Containerizing and Running Locally
1. Verify the Dockerfile
```bash
# Use the `python:3.9` as a source image from the Amazon ECR Public Gallery
# We are not using `python:3.7.2-slim` from Dockerhub because it has put a  pull rate limit.
FROM public.ecr.aws/sam/build-python3.9:latest

# Set up an app directory for your code
COPY . /app
WORKDIR /app

# Install `pip` and needed Python packages from `requirements.txt`
RUN pip install --upgrade pip
RUN pip install -r requirements.txt

# Define an entrypoint which will run the main app using the Gunicorn WSGI server.
ENTRYPOINT ["gunicorn", "-b", ":8080", "main:APP"]

```

2. Store Environment Variables
Create a file named .env_file and save both JWT_SECRET and LOG_LEVEL into that .env_file
```bash
JWT_SECRET='myjwtsecret'
LOG_LEVEL=DEBUG
```

3. Start the Docker Desktop service.

4. Build an image
```bash
docker build -t myimage .
```

Result:
![Docker build](images/docker_1.png)

5. Create and run a container (named is myContainer1)
```bash
docker run --name myContainer1 --env-file=.env_file -p 80:8080 myimage
```

Result:
![Docker run](images/docker_2.png)

6. Check the endpoints
![check healthy app](images/docker_3.png)

![check auth, content apis](images/docker_4.png)


### Deploy application to Kubernetes Cluster
1. CloudFormation Design
![CloudFormation Design](images/ek_1.png)

2. Create EKS Cluster
```bash
eksctl create cluster --name simple-jwt-api --nodes=2 --instance-types=t2.medium --region=us-east-2
```

- get cluster information
![cluster information](images/ek_2.png)

3. get Current AWS account
![Current AWS account](images/ek_3.png)

4. get current configmap
![current configmap](images/ek_4.png)

5. Get JWT_SECRET
![JWT_SECRET](images/ek_5.png)

6. Get Github access token
refer: https://github.com/settings/tokens/

7. Create CloudFormation Stack
- edit: ci-cd-codepipeline.cfn.yml (refer to source code)
- create stack
![create stack](images/ek_6.png)

8. create pipeline
refer buildspec.yml file in source code

![pipeline](images/ek_7.png)

### Test Endpoint
- Run this command to get "simple-jwt-api" service information
```bash
kubectl get services simple-jwt-api -o wide
```

![alt text](images/ek_8.png)

-/auth endpoint
```bash
export TOKEN=`curl -d '{"email":"thuy123@email.com","password":"abc123"}' -H "Content-Type: application/json" -X POST aff937e5681b84284941e9ea234294f9-1482688595.us-east-2.elb.amazonaws.com/auth  | jq -r '.token'`
```

![alt text](images/ek_9.png)

- /contents endpoint
```bash
curl --request GET 'aff937e5681b84284941e9ea234294f9-1482688595.us-east-2.elb.amazonaws.com/contents' -H "Authorization: Bearer ${TOKEN}" | jq
```

![alt text](images/ek_10.png)

### Delete CloudFormation Stack
![alt text](images/ek_11.png)

### Delete EKS Cluster
```bash
eksctl delete cluster simple-jwt-api  --region=us-east-2
```

### Delete parameter-store
```bash
aws ssm delete-parameter --name JWT_SECRET
```
