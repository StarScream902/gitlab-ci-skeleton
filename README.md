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
