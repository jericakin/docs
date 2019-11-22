## Defining Goal Sets

SDM Goal Sets can be defined with a special YAML format or alternatively using
JavaScript. Here we are going to focus on YAML-based definitions. Later in this
guide we'll return to defining goal sets and show how this can be accomplished
using code.

### Introduction to the YAML syntax

The YAML-based declarations used to define the SDM goal sets consist of _goal
contributions_ that combine push tests and goals. Let's revisit a previous example:

```yaml
node_build:

  test:
  - ...

  depends_on:
  - ...

  goals:
  - ...
```

This defines a goal contribution called `node_build` with a `test` and `goals`
property. The name of the goal contribution can be any valid YAML key. It should
be something that describes the goals that would activate if the contribution's test
would pass.

Additionally this introduces the optional `depends_on` property to describe ordering
between goals.

#### `<name>.test`

`test` is optional and takes one of more push tests. Refer to [Using Push Tests]()
for more details on how to use push tests.

If `test` specifies a list of push tests, those are logically combined using `and`
logic.

#### `<name>.goals`

`goals` takes one or more goals. Refer to [Using container goals]() for more details
on how to define goal instances using Docker containers.

If `goals` specifies an array of goals, those goals are executed in the order they
are defined.

To execute goals in parallel, goals can be nested in arrays. Consider the following
example:

```yaml
node_build:

  goals:
  -
    - containers:
      - name: npm test
        image: node:8-slim
        args:
        - sh
        - -c
        - npm run compile --if-present
    - containers:
      - name: npm lint
        image: node:8-slim
        args:
        - sh
        - -c
        - npm run lint --if-present
  - containers:
    - name: npm test
      image: node:8-slim
      args:
      - sh
      - -c
      - npm test
```

This example nests the two container goals for running `npm compile` and `npm
run lint` in an additional array which will execute them in parallel. After
those two goals finish, `npm test` will execute as it is defined in the root
array of goals.

#### `<name>.depends_on`

`depends_on` is optional and can be take one or more names of goal contributions
that the goals of this contribution should depend on. Depending on goals means
that if those goals are part of the final goal set, they have to finish before
any of the current goals would start.

Consider the following example:

```yaml
 node_build:

  test:
  - has_file: package.json

  goals:
  - containers:
    - name: npm test

mvn_build:

  test:
  - has_file: pom.xml

  goals:
  - containers:
    - name: mvn package
      ...

docker_build:

   depends_on:
   - node_build
   - mvn_build

   goals:
   - containers:
     - name: docker build

```

`docker_build` expresses a dependency to both `node_build` and `mvn_build`.
Practically only one of `npm test` or `mvn package` will end up in the final
goal set. Regardless of which of those two goals ends up being activated,
`docker build` will only run once the previous build goal is completed.

## Activating Goals

Because an SDM can react to events across all of our Git repositories and
other connected resources, it is important to be able to select which
goals need to be activated for a specific event. 

For example, a Git push to a project with a Maven `pom.xml` likely requires
a Maven build to occur and should skip any NPM related goals. Furthermore
repositories that have a Dockerfile should probably go through a Docker 
build step; for repositories without a Dockerfile, the Docker build goal
should be skipped or rather not activated. 

Which goals to activate is controlled by _Push Tests_ and _Goal Tests_. 

### Push Tests

A _push test_ allows the owner of an SDM to define predicates against
a project or Git push to determine if a goal should be activated. 

Examples for such push tests include:

 * check the name of branch against a list of configured names
 * check if the project has a `pom.xml` or `Dockerfile`
 * verify that the change being pushed is more than a documentation change
 
The following pre-defined push tests are available to be used:

#### `has_file`

The `has_file` push test takes a file path to verify if the provided file
exists in the repository that is being pushed to.

```yaml
node_build:
  test:
  - has_file: package.json
```

#### `has_file_containing`

`has_file_containing` can be used to check files for the existence of certain
content. This push test takes two properties: `pattern` to define a glob
pattern for the files to match and `content` which should be a regular 
expression to check the file content.

The following shows an example that checks if the `package.json` contains
a reference to the NPM packages `mongoose` or `connect-mongo`.

```yaml
node_build:
  test: 
  - has_file_containing: 
      pattern: package.json
      content: mongoose|connect-mongo
``` 

#### `is_branch`

`is_branch` validates the branch of the Git push against a provided branch
name or regular expression.

In the following example only pushes to `feature-*` branches would activate
the `npm test` goal.

```yaml
node_build:
  test:
  - is_branch: ^feature-.*$
  goals:
  - atomist/npm-goal/test@master
```

#### `is_default_branch`

This push test is useful to test if a push occurs on the repositories 
default branch.

```yaml
node_build:
  test:
  - is_default_branch
  goals:
  - deploy
```

#### `is_material_change`

A material change is a concept that can used to determine if a push contains
changes that require the execution of a full CI/CD pipeline. For example
changes to the repository's `README.md` or documentation changes shouldn't 
trigger a full build, test and deployment cycle. 

```yaml
immaterial:
  test:
  - not:
      is_material_change:
        files:
        - package.json
        - Dockerfile
        extensions:
        - ts
  goals:
  - lock
```

The `is_material_change` push tests takes these properties:

  * `extensions`: file extensions to watch for; without .
  * `files`: file paths to watch for
  * `directories`: directory paths to watch for
  * `patterns`: glob patters to watch for

More on `not` and the `lock` goal further down in this documentation.

### Combination of Push Tests

Often times it is convienent to combine push tests with logical primitives
like `and`, `or` or `not`. 

#### `and`

The `test` key of a goal contribution takes an array of push tests. This 
equvilant to wrapping the push tests with an `and`.

The following snippet

```yaml
node_build:
  test:
  - has_file: package.json
  - has_file: Dockerfile
```

can be re-written to:

```yaml
node_build:
  test:
  - and:
    - has_file: package.json
    - has_file: Dockerfile
```

#### `or'

When wrapping push tests with `or`, only one push test has to evaluate to 
`true` to make the entire test pass.

```yaml
build:
  test:
  - or:
    - has_file: package.json
    - has_file: pom.xml
```

This test would pass if the project either has a `package.json` or `pom.xml`
or both.

#### `not`

`not` negates the provided push test. Please note that `not` only accepts one
push test.

```yaml
mvn_build:
  test:
  - not:
    has_file: package.json
```

This test would evaluate to `true` only for repositories that don't have a
`package.json`.

### Goal Tests

Goal Tests can be used to activate goals on, eg other goals finishing or
failing. Those goals can origniate from different or the same SDM. This
allows to build connected networks of goals and SDMs. 

For example an organization might have two or more SDMs that are responsible
for executing stack or language specific build and test goals. Once those
build goals finish another SDM can activate a goal to build a Docker image,
followed by a deployment SDM adding the staging and production deployment
goals.

#### `is_goal`

```yaml
docker_build:
  test:
  - is_goal: 
    name: ^build.*$
    state: success
    test:
      has_file: Dockerfile
```

The above example declares a goal test that matches every goal that finishes
successfully and its name is prefixed with `build`. Additionally a nested push
test can be used to further assert the project on which that goal occured. Here
we only want to activate the Docker build goal for repositories that have a
`Dockerfile`. 

## Defining Goals

Goals are the activities you want to run on your projects or repositories; goals
can do things like running a build or executing tests on your projects sources.
They can also run linters or automatically update the changelog once you release
a new version of your project. 

For many of these tasks there are tools developers use every day. Most -if not
all- are available from Docker images in public Docker registries like Docker
Hub. 

With an SDM you can use Docker images to back a goal; we call that a container
goal. A container goal allows the SDM author to define common input for running
Docker containers like environment variables, command and arguments and secrets.

To allow running containers use other serivces like e.g. a Mongo database for 
integration tests, a container goal can actually define more than one container. 

Atomist and the community provide a catalog of pre-defined container goals 
that can be referenced in your SDM goal contributions to make defining goals
and resuing goals across your organization very easy.

Lastly, goals can also be implemented in JavaScript or TypeScript and embedded
in an SDM. Refer to [Using JavaScript to create a Container Goal]() to more on
creating goals in code.

### Defining a Container Goal

The following YAML snippet defines a container goal with two containers; one
Node.JS container running `npm ci && npm test` and a second container making 
a Mongo database available. 

The Mongo database container is only created for
repositories that have a dependency to `mongoose` or `connect-mongo` in their
`package.json`. 

This once again shows the fact that an Atomist SDM can operate and activate
delivery goals across your entire organization; normally the Mongo database
requirement would be hard-coded in the repository's build pipeline. 

```yaml
node_build:  
  
  goals:

  - containers:
    - name: npm-test
      image: node:8-slim
      env:
      - name: NODE_ENV
        value: development
      - name: BLUEBIRD_WARNINGS
        value: '0'
      args:
      - sh
      - -c
      - npm ci && npm test
    - name: mongo
      image: mongo:3.6
      test:
      - has_file_containing:
          glob_pattern: package.json
          content: mongoose|connect-mongo
  input:
  - version
  output:
  - classifier: cache
    pattern:
      directory: node_modules
```

The following sections will document the various properties of a Container Goal.

#### `containers[].name`

Required name of the container.

#### `containers[].image`

Required Docker image name of the container.

#### `containers[].command`

Optional Docker image entrypoint.

#### `containers[].args`

Optional Docker command and arguments.

#### `containers[].env`

Optional environment variables to set in Docker container.

#### `containers[].ports`

Optional ports to expose from container.

#### `containers[].volume_mounts`

Optional volumes to mount in container.

A volume mount needs to have a `name` and `mount_path` key. The `name` referes
to the name of `volume` and `mount_path` is the location the volume should be
mounted inside the container.

#### `containers[].test`

A push test that enables the container goal only when the push test evaluates
to `true` for the current repository. 

#### `containers[].secrets`

`secrets` is a way to declare secret requirements and how those requirements
should be fulfilled. Please refere to [Using Secrets]() for more on secrets.

#### `containers.approval`

Sets the goal to require approval after executing.

#### `containers.pre_approval`

Sets the goal to require a pre-approval before it executes.

#### `containers.retry`

Sets the goal to allow retry in case of failure.

### Environment Variables
### Using Secrets
### Placeholders in YAML
### Using pre-defined Goals


