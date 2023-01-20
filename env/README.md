# ENV at NIRV.ai

- when 12factor isnt enough

## TLDR

- if services check $ENV for consuming, locating (...etc) a $VAR, your doing something wrong

## supported environments

### validation

- as a developer with cloud certification, nothing beats development in pure docker
- [left-shifting](https://www.dynatrace.com/news/blog/what-is-shift-left-and-what-is-shift-right/) platform responsibilities into development IMO is irresponsible and unrealistic
  - you could say `validation` is the furthest west thats appropriate
- validation goals:
  - takes local docker services as input and outputs cloud-ready services
  - first round of obfuscation

### test

- test goals:
  - second round of obfuscation
  - all sorts of technical testing, penetrations and validations

### stage

- stage goals:
  - third round of obfuscation
  - SLA verification
  - all sorts of business validation

### production

- production goals
  - final round of obfuscation
  - party time

## secrets & certs

- secrets
  - prefer vault in all circumstances
  - in memory (12factor)
- TLS
  - buildtime: /etc/ssl/{certs,private}
  - runtime: /run/secrets

```sh

# `cd $JAIL`
# by default permissions come from source, not symlinks
# docker secrets bug: @see https://github.com/docker/compose/issues/9648
sudo chown -R $USER:$USER dev.nirv.ai
sudo chown -R $USER:$USER mad.nirv.ai
sudo chown -R consul:consul mesh.nirv.ai

# docker secrets are sourced from /etc/ssl/certs
# thats why we need the symlinks
sudo mkdir /etc/ssl/certs/{dev,mad,mesh}.nirv.ai
sudo mkdir /etc/ssl/private/{aws,vault}
sudo ln -s `pwd`/dev.nirv.ai/letsencrypt/live/dev.nirv.ai/* /etc/ssl/certs/dev.nirv.ai
sudo ln -s `pwd`/mad.nirv.ai/tls/* /etc/ssl/certs/mad.nirv.ai
sudo ln -s `pwd`/mesh.nirv.ai/tls/* /etc/ssl/certs/mesh.nirv.ai
sudo ln -s `pwd`/aws/* /etc/ssl/private/aws
sudo ln -s `pwd`/vault/*/*/*.json /etc/ssl/private/vault
```
