# GitLab CI skeleton
It is a demonstration repository how to create mono repo wit CI/CD for all your services

## Why i think that it is the best way to build CI/CD for your services
If you have an only couple of services, it is not a big problem, just copy/paste a ci/cd file 
and make some changes after. But if you have a lot of services, most likely they have similar 
steps for building, some of them for the testing too etc. Or you want to add some new steps and 
scale it for all services. This solution keeps your code DRY, you don't have to jump from repo 
to repo if you want to change something.

Yes, it may seem a bit complex, but if you understand this approach you will be more flexible 
and will be able to implement new features faster, change your pipelines faster and even more

## Concept
There are several key elements:
* `ci-entrypoint` - it is like a router for all services for choosing which pipeline it needs to choose
* `pipeline` - here it needs to describe the steps which it needs to execute
* `template` - it is like classes, which it may inherit and change if it needs

## How does it work?

For using the **CI** you have to put the text below in the `.gitlab-ci.yml` file in the root of your repo

```YAML
include:
  - project: 'full/path/to/the/ci'
    ref: '$DEFAULT_CI_PROJECT_REF' # the branch from where you want to take the CI/CD
    file: '/ci-entrypoint.yaml'
```

Then, when your code will be triggered, GitLab will take the pipeline definition from the 
`full/path/to/the/ci` project where all the logic of CI/CD is stored.

The entrypoint of this mechanisme is `/ci-entrypoint.yaml`, there it is described which pipeline 
it needs to choose for the further process

```yaml
include:
  # entrypoint for the 11 JAVA services
  - local: '/pipelines/java/11.yaml'
    rules:
      - if: $CI_PROJECT_NAME == "java-11-app-1" # what the projects have to use this pipeline
      - if: $CI_PROJECT_NAME == "java-11-app-2"
```

In a pipeline, we are already defining what steps will be executed (build, test, deploy, etc) 
and we are describing here exceptions for the specific buildings, tests. Due to the fact that, 
the most steps have the same manipulations, which we may move out in templates and reuse it

```yaml
include:
  - local: '/templates/java/default.yaml'
  - local: '/templates/java/11.yaml'
  - local: '/templates/build_docker/default.yaml'
  - local: '/templates/deploy/deploy_default.yaml'

stages:
  - build
  - test
  - build_docker
  - deploy

build:
  extends: .default_build # use a template which is described in /templates/java/default.yaml

test:
  extends: .default_test # use a template which is described in /templates/java/11.yaml
  except: # make an exception
    variables:
      - $CI_PROJECT_NAME == "java-11-app-1"
      - $SKIP_TESTS != "false"

test_kafka_mongo:
  extends: .test11_kafka_mongo # use a template which is described in /templates/java/11.yaml
  rules:
    - if: $SKIP_TESTS != "false"
      when: never
    - if: $CI_PROJECT_NAME == "java-11-app-1"
```

And finaly the templates themselvs. In them it is necassary to describe steps, which you will reuse 
in your pipelines.

## Deploy
Previews part was about this repo and how does it works

In this part, I'm going to describe especially the deployment part

For example, you have several count of services and they have to be deployed in different way. Therefor, you need to do something like in `ci-entrypoint.yaml`. Describe which services do you want to deploy in one way and which in another in example of `pipelines/java/11.yaml`.

```yaml
include:
  - local: '/templates/java/default.yaml'
  - local: '/templates/java/11.yaml'
  - local: '/templates/build_docker/default.yaml' # this and couple lines above about ci
  - local: '/templates/deploy/deploy_default.yaml' # and this already about deploy
    rules:
      - if: $CI_PROJECT_NAME == "java-11-app-1" # and which services are we going to deploy via helm template
  - local: '/templates/deploy/java_default.yaml' # another way of deployment
    rules:
      - if: $CI_PROJECT_NAME == "java-11-app-2" # and which services are we going to deploy via ansible template
```

And let's see on this two deployment ways, Helm and Ansible

### Helm
`templates/deploy/helm.yaml`
The file which is describing the template for deploying apps via Helm. 
All description will be as a comments
```yaml
.helm_deploy:
  stage: deploy
  image:
    name: my-custom-docker-image:helm-3.6 # with python, boto3 and another stuff for the getting secrets from the aws parameter store
    entrypoint: [ "" ]
  allow_failure: false
  needs: ["build_docker"] # we need a docker image for deploy and therefore we are writing this needs
  before_script:
    - cp $PWD.tmp/${get_parameters_creds} ~/.aws/credentials # I'm using custom python script for getting secrets from amazon parameter store, therefore to I'm preparing creds for the aws
    - git clone --branch=${DEFAULT_CHARTS_REPO_REF} https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/repo-of-my-helm-charts.git ./helm # Then I'm cloning helm charts repo
    - AWS_PROFILE=aws_get_param python3 ./helm/get-parameters.py /${env_name_short}/${CI_PROJECT_NAME} # And executing custom script for getting secrets from amazon parameter store
  script:
    - |
      command="helm upgrade --install \ # variables are defining in the step below
        --kubeconfig $PWD.tmp/${K8S_CONFIG_FILE} \ # kubeconfig for connection to k8s cluster
        --namespace ${K8S_NAMESPACE} \
        ${CI_PROJECT_NAME} ./helm/charts/${CI_PROJECT_NAME} \ chart name and path to the chart
        --set image.repository=${CI_REGISTRY_IMAGE} \ # Which registry it needs to use
        --set image.tag=${DOCKER_IMAGE_TAG} \
        --set def.environment=${CI_ENVIRONMENT_NAME} \
        --values ./helm/charts/${CI_PROJECT_NAME}/values-${env_name_short}.yaml \
        --values ./secrets.yaml \ # secrets file which was generated by script above
        --timeout 5m \
        --atomic"
    - echo $command
    - eval $command
```
You may write the deploy commnd in one line, without this variable `command` and next rows `echo $command`, `eval $command`. I did it only for simplicity of view as a text in VS Code, and print it in output of a pipeline and then execute it.

`templates/deploy/deploy_default.yaml`
Just using template and defining variables you need for deploying a service in particular environment
```yaml
include:
  - local: '/templates/deploy/helm.yaml'

deploy_to_sandbox:
  extends: .helm_deploy
  environment:
    name: sandbox
  variables:
    K8S_CONFIG_FILE: "SANDBOX_K8S_CONFIG_FILE"
    K8S_NAMESPACE: ${CI_ENVIRONMENT_NAME}
    get_parameters_config: "aws_get_parameters_config"
    get_parameters_creds: "aws_get_parameters_creds"
    env_name_short: ${CI_ENVIRONMENT_NAME}
  when: manual
  tags:
    - sandbox
```

### Ansible

`templates/deploy/ansible.yaml`
The file which is describing the template for deploying apps via Ansible
```yaml
.ansible:
  stage: deploy
  image:
    name: my-custom-docker-image:ansible-2.13
    entrypoint: [ "" ]
  allow_failure: false
  variables:
    ANSIBLE_USER_PRIVATE_KEY: "ANSIBLE_USER_PRIVATE_KEY" # for ssh connection
    ANSIBLE_VAULT_PASS: "ANSIBLE_VAULT_PASS"
  before_script:
    - eval $(ssh-agent -s) # preparing ssh for connection
    - mkdir ~/.ssh
    - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    - ssh-add <(echo "$ANSIBLE_USER_PRIVATE_KEY")
    - mkdir ~/.ansible # preparing ansible for work
    - echo $ANSIBLE_VAULT_PASS > ~/.ansible/vault_password
    - git clone --branch=${DEFAULT_ANSIBLE_REPO_REF} https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.com/my-ansible-repo.git ./ansible
    - cd ./ansible
```

`templates/deploy/java-ansible.yaml`
just step with running a deploy playbook
```yaml
include:
  - local: '/templates/deploy/ansible.yaml'

deploy java_ansible:
  extends: .ansible
  needs: ["upload-artifact"]
  when: manual
  script:
    - ansible-playbook playbooks/java_deploy.yaml -i envs/dev/inventory.ini --user ansible-user -e service_name=${CI_PROJECT_NAME} -e service_version=${CI_COMMIT_SHORT_SHA}
  tags:
    - sandbox
```
