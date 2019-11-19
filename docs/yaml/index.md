## Getting Started

With an Atomist Software Delivery Machine you get a powerful, event-
driven and code-oriented platform that lets you automate your 
organization's software delivery processes and policies beyond CI/CD
pipelines. 

### About the SDM

An SDM is a process that runs in your environment and serves as the
scheduler and host of your delivery goals. **Goals** are the steps 
or jobs that you want to activate when a certain event, like a Git 
push - occurs. 

To determine which goals to schedule on a particular Git push, the 
SDM introduces the concept of **push tests**.

All goals that pass push tests are complied into a **goal set** and
passed for execution to the SDM's goal schedulers.

The three core concepts of an SDM:

#### Goal

A goal encapsulates the smallest unit of work you want to activate on 
a certain Git push. Examples could be tasks like running a Maven or 
Gradle build or deploying a Docker image into a Kubernetes cluster. 

Goals can be backed by existing, 3rd party-provided Docker images or 
your own Docker images as well as coded directly into the SDM in
JavaScript or TypeScript.  

#### Push Test

A push test is a way to condition goals to only get scheduled for 
certain types of repositories, pushes or other aspects. 

Examples could be testing for the existence of a Dockerfile to 
activate the Docker build goal, or making sure deployment only happens
for your repository's default branch.

#### Goal Set 

Once push tests are evaluated, all determined goals are collected
into a goal set.

Different SDMs can create independent Goal Sets —eg to add security
scanning to every Git push— or can work together to orchestrate large 
delivery networks on your repositories.    

### Create your Atomist Workspace

Before you can create and run your first SDM, you need to create an 
Atomist workspace. 

`< copy the getting started guide >`

and install the Atomist CLI.

### Creating your first Goal Set

Later in this documentation you'll see how to create a Goal Set in
JavaScript. But for those first examples we intent to keep things as 
simple as possible and define the Goal Set in YAML and reuse some 
existing Docker images.

#### Java with Maven 

The following YAML snippet defines a goal contribution called `mvn_build`
that has one goal `containers` which will run one Docker image. Specifically 
this will run `mvn package` using the standard `maven` Docker image from 
Docker Hub on every Git push. 

```yaml
mvn_build:

  goals:
  - containers:
    - name: mvn test
      image: maven:3-jdk-8-slim
      args:
      - mvn
      - package
```

Note: The `containers` key already indicates that you can run multiple 
Docker images as part of one goal. This is useful to add side car 
containers like databases etc. to the execution of the main goal container.

Because there are no push tests defined in this goal contribution, the `goals`
will be activated on each push. 

#### Node with NPM

The following sample adds a 2nd container to run a Mongo database along-side
a `npm test` container. 

There is also now a push test called `has_file` that checks for the existence
of a file called `package.json` to only activate the `npm test` goal when the
Git push is for a NPM project. 

```yaml
node_build:  
  
  test: 
  - has_file: package.json
     
  goals:
  - containers:
    - name: npm test
      image: node:8-slim
      env:
      - name: MONGODB_URI
        value: mongodb://mongo:27017/test
      args:
      - sh
      - -c
      - npm install && npm test
    - name: mongo
      image: mongo:3.6
  output:
  - classifier: dependencies
    pattern:
      directory: node_modules
```

Additionally the goal used in this sample, declares that it wants the `node_modules`
directory cached under the key `dependencies`. This allows later goals to refer to the
contents of the `node_modules` directory via the `dependencies` cache key.

### Running the SDM locally

Starting the SDM from the previous section, we'll use the Atomist 
CLI and run:

```shell script
$ atomist start --repository-url git@github.com:atomist/samples.git \
      --yaml lib/sdm/yaml/maven-sdm.yaml
```

or

```shell script
$ atomist start --repository-url git@github.com:atomist/samples.git \
      --yaml lib/sdm/yaml/npm-sdm.yaml
```

This command will clone the samples repository and start the SDM from the
`lib/sdm/yaml/maven-sdm.yaml` or `lib/sdm/yaml/npm-sdm.yaml` YAML file.
