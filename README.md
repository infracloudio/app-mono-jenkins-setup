# Monorepo - Kubernetes


This is a working example of using Monorepo and deploying to Kubernetes using Helm. For details check the post:  http://www.infracloud.io/monorepo-ci-cd-helm-kubernetes

There are 5 repositories involved:

- [Application Source Code](https://github.com/infracloudio/app-mono)
- [Helm charts for applications](https://github.com/infracloudio/app-mono-helmcharts)
- [Helm values/state for applicatiopns](https://github.com/infracloudio/app-mono-helmstate)
- [Orchestrator job](https://github.com/infracloudio/app-mono-orchestrator)
- [Jenkins Setup for end to end example - this repo](https://github.com/infracloudio/app-mono-jenkins-setup)


## Setup Instructions

### 1. Kubernetes  cluster setup

- Initialize Helm and install Tiller into kubernetes cluster using `helm init`
- Find tiller-token secret name - `kubectl get secret -n kube-system | grep tiller-token`
- Find SA token from tiller-token secret using `kubectl describe secret -n kube-system $TILLER_TOKEN_SECRET_NAME`

### 2. Set environment variables in Jenkinsfiles

You can store sensetive credentials in jenkins credential store and inject it in jenkins as given [here](https://jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials) 

a. For kubernetes cluster access, set following environment variables [in Jenkinsfile in app-mono-helmstate with correct details](https://github.com/infracloudio/app-mono-helmstate/blob/master/Jenkinsfile)

- K8S_SERVER = "K8S_SERVER_ADDRESS" (Can be found in kubeconfig file from where you can access the cluster)
- K8S_TILLER_TOKEN = "TILLER_SERVICE_ACCOUNT_TOKEN" (Got it from Step 1)
- K8S_CA_BASE64 = "BASE_64_ENCODED_CA_CERT" (can be found in kubeconfig file)
- GITHUB_HOOK_SECRET = webhook secret which set in github webhook url

b. For github access, set following variables in [Jenkinsfile in app-mono-orchestrator repo](https://github.com/infracloudio/app-mono-orchestrator/blob/master/Jenkinsfile)

- HELM_STATE_GIT_REPO = git repo url where helm values.yamls are stored
- BRANCH = git repo branch where helm values.yamls are stored
- HELM_STATE_REPO = directory name of the helm state repo
- DOCKERHUB_HOOK_SECRET = webhook secret which was set in dockerhub webhook url (must be different that what has been set in app-mono-helmstate/Jenkinsfile)
- GIT_USR = Github username
- GIT_PSW = Github password

c. To build and push docker image for app-mono applications, set following env vars in [Jenkinsfile of app-mono repo](https://github.com/infracloudio/app-mono/blob/master/Jenkinsfile)

- GITHUB_HOOK_SECRET = webhook secret which set in github webhook url
- DOCKERHUB_PSW = dockerhub password
- DOCKERHUB_USR = dockerhub username

### 3. Setup webhooks

a. Set webhook url in github repo [app-mono to trigger](https://github.com/infracloudio/app-mono/settings/hooks) image build job

- Set url as: http://JENKINS_HOST/generic-webhook-trigger/invoke?token=$GITHUB_HOOK_SECRET (set in app-mono/Jenkinsfile)
- Select content type as "application/json"

b. Set webhook url in github repo app-mono-helmstate to [trigger deployment job](https://github.com/infracloudio/app-mono-helmstate/settings/hooks)
- Set url as: http://JENKINS_HOST/generic-webhook-trigger/invoke?token=$GITHUB_HOOK_SECRET (set in app-mono-helmstate/Jenkinsfile)
- Select content type as "application/json"
- NOTE: GITHUB_HOOK_SECRET set in webhook url must be different than the one set for app-mono repo

c. Set webhook url in Dockerhub for triggerting orchestrator job. URL format is: http://JENKINS_HOST/generic-webhook-trigger/invoke?token=$GITHUB_HOOK_SECRET (set in app-mono-helmstate)

### 4. Jenkins setup:
   - Prerequisite:
    - install following plugins
     - generic-webhook-trigger
     - github
     - job-dsl
     - credentials
     - credentials-binding (optional)

a. If you are running jenkins on docker container, make sure that docker is installed in jenkins image and jenkins user is able to execute docker commands inside the container
b. Create github api token to access private repositories ([https://github.com/settings/tokens](https://github.com/settings/tokens))
c. Create jenkins credentials with ID "github-credentials"
d. Goto to Credentials >> Add Credentials >> Select Kind as "Username and password" >> Add Username and Password >> Set ID "github-credentials" >> OK

### 3. Creating base DSL job which creates other jobs

a. Create New Item as "Freestyle-project"
b. Select "Git" in "Source code management" section. Set repository URL "[https://github.com/infracloudio/app-mono-jenkins-setup.git](https://github.com/infracloudio/app-mono-jenkins-setup.git)"
c. In Build section, add build step "Process Job DSLs" and provide DSL Scripts path "job_dsl.groovy"
d. Save and build this job

4. If the job fails with msg "ERROR: script not yet approved for use", goto Manage Jenkins >> In process Scipt Approval >> Click Approve and rerun the job

5. Once the job succeed, you can see 3 new jobs created on jenkins portal
    - app-mono-build-job
    - image-orchestrator-job
    - k8s-deploy

6. Trigger all the 3 generated jobs manually so that they fetch Jenkinsfile from the github repo and configure webhook triggers (this is one time process). The builds are expected to fail since they are not triggered by webhooks

7. Now your CI/CD workflow is ready.
    
- Any commit to the app-mono repo will trigger image build job (app-mono-build-job), depending upon files changes, it will build app-mono-www or app-mono-api or both images and will push images to dockerhub
- On dockerhub image push event, image-orchestrator-job will be triggered which will update values.yaml in app-mono-helmcharts repo and commit the changes
- Now, when values.yaml changes get committed to app-mono-helmstate repo, it calls the webhook url and triggers k8s-deploy job which will deploy the application to k8s cluster
