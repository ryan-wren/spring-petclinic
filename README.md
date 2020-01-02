
# Spring PetClinic Sample Application
 
TODO: add CircleCI build status

TODO: add link to Spring PetClinic page on Spring

## Sample configurations
[A basic build](#a-basic-build)

[Splitting tests across parallel containers](#splitting-tests-across-parallel-containers)

### A basic build
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

### Storing code coverage artifacts
```yaml
version: 2.0

jobs:
  test:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - run: ./mvnw test verify
      - store_artifacts:
          path: target/site/jacoco/index.html

workflows:
  version: 2

  test-with-store-artifacts:
    jobs:
      - test
```
The Maven test runner with the JaCoCo plugin generates a code coverage report during the build. To save that report as a build artifact, use the `store_artifacts` step.

### Caching dependencies
```yaml
version: 2.0

jobs:
  build:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "pom.xml" }} # appends cache key with a hash of pom.xml file
            - v1-dependencies- # fallback in case previous cache key is not found
      - run: ./mvnw -Dmaven.test.skip=true package
      - save_cache:
            paths:
              - ~/.m2
            key: v1-dependencies-{{ checksum "pom.xml" }}
```
The first time I ran this build without any dependencies cached (https://circleci.com/gh/annapamma/spring-petclinic/45), it took 2m14s. Once I was able to just restore my dependencies, the build took 39 seconds (https://circleci.com/gh/annapamma/spring-petclinic/46). 

Note that the `restore_cache` step will restore whichever cache it first matches. I add a restore key here as a fallback. In this case, even if pom.xml changes, I can still restore the previous cache. This means my job will only have to fetch the dependencies that have changed between the new pom.xml and the previous cache. 

### Splitting tests across parallel containers
```yaml
version: 2.0

jobs:
  test:
    parallelism: 2 # parallel containers to split the tests among
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
      - store_test_results: # We use this timing data to optimize the future runs
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
Splitting tests by timings is a great way to divide time-consuming tests across multiple parallel containers. 
I think of splitting by timings as requiring 4 parts:
1) a list of tests to split
2) the command: circleci tests split --split-by=timings
3) containers to run the tests
4) historical data to intelligently decide how to split tests

To collect the list of tests to split, I simply pulled out all of the Java test files with this command: `circleci tests glob "src/test/**/**.java"`. 
Then I used `sed` and `tr` to translate this newline-separated list of test files into a comma-separated list of test classes. 

Adding `store_test_results` enables CircleCI to access the historical timing data for previous executions of these tests, so the platform knows how to split tests to achieve the fastest overall runtime. 
