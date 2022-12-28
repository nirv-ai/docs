# nirvai scripts

> CLI-as-a-Service

- guides for writing cross-platform, resiliant posix compliant shell scripts.
- nirvai scripts is a single interface to all CLIs supporting NIRVai platform components and packages
- [source code](https://github.com/nirv-ai/scripts)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## INSTALLATION

- wherever you've closed this repo, add the path to your init file

```sh
# if `pwd` === home/poop/git/private/nirv/scripts
# update your ~/.bashrc
export PATH="/home/poop/git/private/nirv/scripts:$PATH"

```

## best practices

### never use a CLI directly, always create a wrapper

- platform wide upgrades: move to the latest version and update in a single place
- useful side effects: wouldnt it be great to run fmt, check, and validate on every nomad cmd?
- standardized interfaces
- CLIs as a service

### always use #!/usr/bin/env [SHELL]

- even for sh, so consumers can use whatever shell by pointing the shebang to the shell of their choosing

## available scripts

- [hashicorp/vault](../vault/README.md)
- [docker/compose/refresh](../docker/README.md)
- [docker/compose/reset](../docker/README.md)
- [docker/container/exec](../docker/README.md)
- [docker/registry](../docker/README.md)
- [hashicorp/nomad](../nomad/README.md)
