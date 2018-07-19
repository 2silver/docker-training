# Linting


## Linting

!!! abstract

    Best practices for linting `Dockerfiles`.

We use [hadlolint](https://github.com/hadolint/hadolint) for linting.

One other nice linter is [dockerfile-lint](https://github.com/projectatomic/dockerfile_lint), currently this one is not working with multi-stage build images.

### Usage

#### Docker

``` console
docker run --rm -i hadolint/hadolint hadolint - < Dockerfile
```

For more, like using it with Atom, Sublime, etc, please see the [docs](https://github.com/hadolint/hadolint).

## Continuous Integration

The [docs](https://github.com/hadolint/hadolint) are showing examples about Travis and Gitlab.

### CircleCI.

Adjust your `config.yml`:

``` yaml
jobs:
  "lint dockerfile":
    docker:
      - image: hadolint/hadolint:latest-debian
    <<: *defaults
    steps:
      - checkout
      - run:
          name: Runnig linter against Dockerfile
          command: |
            cd dockerfiles
            hadolint Dockerfile
```

As you can see in the example above, our `Dockerfiles` are located in the *dockerfiles* directory.

Make sure to adjust the *workflow* part, too.

``` yaml
workflows:
  version: 2
  docs:
    jobs:
      - "lint dockerfile"
      - "generate docs":
           requires:
             - "lint dockerfile"
```
