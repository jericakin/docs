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
changes that require the execution of a full CI/CD pipeline. For example changes
to the repository's `README.md` or documentation changes shouldn't trigger a
full build, test and deployment cycle.

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

### Goal Tests

## Defining Goals
### Container Goal
### Environment Variables
### Using Secrets
### Placeholders in YAML

## Creating custom Container Goals
### Container Goal Contract
### Creating your first Container Goal
### Testing your Container Goal
### Using JavaScript to create a Container Goal

## Extending the SDM with JavaScript
### Push Tests
### Goals
### Commands
### Events
### Testing Goal Sets
