# podman-quay-cache-buildkite-plugin

A [Buildkite plugin](https://buildkite.com/docs/agent/v3/plugins) to cache Podman images in Quay.

This allows you to define a Dockerfile for your build-time dependencies without worrying about the time it takes to build the image. It allows you to re-use entire container images without worrying about layer caching, and/or pruning layers as changes are made to your containers.

## Example

### Basic usage

```dockerfile
FROM bash

RUN echo "my expensive build step"
```

```yaml
steps:
  - command: 'echo wow'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0
      - KennaSecurity/podman#master
```

### Caching npm packages

This plugin can be used to effectively cache `node_modules` between builds without worrying about layer cache invalidation. You do this by hinting when the image should be rebuilt.

```dockerfile
FROM node:10-alpine

WORKDIR /workdir

COPY package.json package-lock.json /workdir

# this step downloads the internet
RUN npm install
```

```yaml
steps:
  - command: 'npm test'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          cache-on:
            - package-lock.json
      - KennaSecurity/podman#master:
          volumes:
            - /workdir/node_modules
```

The `cache-on` property also supports Bash globbing with `globstar`:

```yaml
steps:
  - command: 'npm test'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          cache-on:
            - '**/package.json' # monorepo with multiple manifest files
            - yarn.lock
      - KennaSecurity/podman#master:
          volumes:
            - /workdir/node_modules
```

### Using another Dockerfile

It's possible to specify the Dockerfile to use by:

```yaml
steps:
  - command: 'echo wow'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          dockerfile: my-dockerfile
      - KennaSecurity/podman#master
```

The subdirectory containing the Dockerfile is the path used for the build's context.

### Specifying a target step

A [multi-stage build] can be used to reduce an application container to just its runtime dependencies. However, this stripped down container may not have the environment necessary for running CI commands such as tests or linting. Instead, the `target` property can be used to specify an intermediate build stage to run commands against:

[multi-stage build]: https://docs.docker.com/develop/develop-images/multistage-build/

```yaml
steps:
  - command: 'cargo test'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          target: build-deps
      - KennaSecurity/podman#master
```

### Specifying build args

[Build-time variables] are supported, either with an explicit value, or without one to propagate an environment variable from the pipeline step:

[build-time variables]: https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg

```dockerfile
FROM bash

ARG ARG_1
ARG ARG_2

RUN echo "${ARG_1}"
RUN echo "${ARG_2}"
```

```yaml
steps:
  - command: 'echo amaze'
    env:
      ARG_1: wow
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          build-args:
            - ARG_1
            - ARG_2=such
      - KennaSecurity/podman#master
```

Additional `podman build` arguments be passed via the `additional-build-args` setting:

```yaml
steps:
  - command: 'echo amaze'
    env:
      ARG_1: wow
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          additional-build-args: '--ssh= default=\$SSH_AUTH_SOCK'
      - KennaSecurity/podman#master
```

### Specifying a repository name

The plugin pushes and pulls images to and from a Quay repository named `buildkite-cache/${BUILDKITE_ORGANIZATION_SLUG}/${BUILDKITE_PIPELINE_SLUG}`. You can optionally use a custom repository name:

```yaml
steps:
  - command: 'echo wow'
    plugins:
      - KennaSecurity/podman-quay-cache#v1.0.0:
          repo-name: my-unique-repository-name
      - KennaSecurity/podman#master
```

## Design

The plugin derives a checksum from:

- The argument names and values specified in the `build-args` property
- The files specified in the `cache-on` and `dockerfile` properties

This checksum is used as the image tag to find and pull an existing cached image from Quay, or to build and push a new image for subsequent builds to use.

## Tests

To run the tests of this plugin, run:

```bash
podman-compose run --rm tests
```

## License

MIT (see [LICENSE](LICENSE))
