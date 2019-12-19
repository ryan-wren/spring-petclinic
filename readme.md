
# Spring PetClinic Sample Application [![Build Status](https://travis-ci.org/spring-projects/spring-petclinic.png?branch=master)](https://travis-ci.org/spring-projects/spring-petclinic/)

TODO: add link to Spring PetClinic page on Spring

## Sample configurations
### The most basic build
```yaml
version: 2.0

jobs:
  build:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: ./mvnw package
```

Version 2.0 configs without workflows will look for a job entitled `build` and run that job in lieu of a workflow.
A job is a essentially a series of commands run in a clean execution environment. Notice the two primary parts of a job: the executor and steps. 

### Using a workflow to test then build
```yaml
version: 2.0

jobs:
  test:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: ./mvnw test

  build:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: ./mvnw -Dmaven.test.skip=true package

workflows:
  version: 2

  test-then-build:
    jobs:
      - test
      - build:
          requires:
            - test

```
A workflow is a dependency graph of jobs. This basic workflow runs a `test` job and a `build` job. 
The `build` job will not run unless the `test` job exits successfully. 

### Splitting tests across parallel containers
```yaml
version: 2.0

jobs:
  test:
    parallelism: 2 # configure the containers to run in parallel
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: |
          ./mvnw \
          -Dtest=$(for file in $(circleci tests glob "src/test/**/**.java" \
          | circleci tests split --split-by=timings); \
          do basename $file \
          | sed -e "s/.java/,/"; \
          done | tr -d '\r\n') \
          -e test
      - store_test_results:
          path: target/surefire-reports

  build:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: ./mvnw -Dmaven.test.skip=true package

workflows:
  version: 2

  test-then-build:
    jobs:
      - test
      - build:
          requires:
            - test
```
