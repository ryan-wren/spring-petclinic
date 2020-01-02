
# Spring PetClinic Sample Application
 
[![CircleCI](https://circleci.com/gh/annapamma/spring-petclinic.svg?style=svg)](https://circleci.com/gh/annapamma/spring-petclinic)

This is an example application showcasing how to run a Java app on CircleCI 2.1. This application uses the [Spring PetClinic sample project](https://projects.spring.io/spring-petclinic/).

This readme includes pared down sample configs for different CircleCI features, including workspace, dependency caching, and parallelism. 
These configs use features from CCI version 2.1, including reusable commands and executors.

To see how we would use orbs to checkout, build, and test in one line, please see the [2.1-orbs-config branch](https://github.com/annapamma/spring-petclinic/tree/2.1-orbs-config).

To see these features in a version 2.0 config, please see the [master branch](https://github.com/annapamma/spring-petclinic/tree/master).

## Sample configurations: version 2.0
- [Building and testing with reusable cache commands](#building-and-testing-with-reusable-cache-commands)
- [Using a workflow to test then build](#using-a-workflow-to-test-then-build)
- [Caching dependencies](#caching-dependencies)
- [Splitting tests across parallel containers](#splitting-tests-across-parallel-containers)
- [Storing code coverage artifacts](#storing-code-coverage-artifacts)
- [All together](#all-together)

### Building and testing with reusable cache commands
```yaml
version: 2.1

commands:
  restore_cache_cmd:
    steps:
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "pom.xml" }}
            - v1-dependencies-
  save_cache_cmd:
    steps:
      - save_cache:
          paths:
            - ~/.m2
          key: v1-dependencies-{{ checksum "pom.xml" }}

jobs:
  test:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - restore_cache_cmd
      - run: ./mvnw test
      - save_cache_cmd

  build:
    docker:
      - image: circleci/openjdk:stretch
    steps:
      - checkout
      - restore_cache_cmd
      - run: ./mvnw -Dmaven.test.skip=true package
      - save_cache_cmd

workflows:
  build-then-test:
    jobs:
      - build
      - test:
          requires:
            - build
```
Notice that the `restore_cache_cmd` and `save_cache_cmd` commands that we have declared at the beginning of the config have been reused in both the `test` and `build` jobs.

