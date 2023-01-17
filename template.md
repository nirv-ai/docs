# NIRVai VAULT

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/YOUR-SERVICE)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/YOUR-SERVICE)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## NIRVai Mindset

> if it doesnt copypasta, it doesnt belong in your stack

insert video

## WHY YOUR-SERVICE ?

- BACKGROUND INFO
- our goal is to achieve:
  - YOUR GOALS

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> YOUR CATCH PHRASE

## Setting up YOUR-SERVICE

### REQUIREMENTS

> if your directory structure _does not_ match below
> modify the inputs in the `# INTERFACE` section that follows

```sh

######################### REQUIREMENTS
# curl        @see https://curl.se/docs/manpage.html
# jq          @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts             # @see https://github.com/nirv-ai/scripts
├── configs             # @see https://github.com/nirv-ai/configs
├── ${REPO_DIR_NAME}
├── secrets             # chroot jail, a temporary folder or private git repo


```

### INTERFACE

```sh
######################### platform interface
# all nirv scripts use this same interface
# LIST THE PLATFORM ENVS YOU USE


######################### YOUR-SERVICE INTERFACE
# LIST YOUR PUBLIC INTERFACE

```

### YOUR-SERVICE STEP 1

- if there are logical steps to setup your app, create a section for each

#### next steps

- Congrats! you have a zero trust development environment backed by YOUR-SERVICE!
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

##### copypasta: YOUR-SERVICE initialization

- every service requires copypasta

```sh

```

##### copypasta: validation

- every service requires validation

```sh

```
