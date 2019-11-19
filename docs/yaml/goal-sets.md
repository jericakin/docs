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

### Using Container Goals
### Using Push Tests
### Activating Goals with Goal Tests
### Environment Variables
### Using Secrets
### Placeholders in YAML
