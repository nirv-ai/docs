# ENV at NIRV.ai

- when 12factor isnt enough

## TLDR

- if services check $ENV for consuming, locating (...etc) a $VAR, your doing something wrong

## architecture

- container consumption
  - everything is stored in a flat directory under /run/secrets/...
  - no matter how it looks in /secrets or /etc/ssl/certs, it must be a flat directory in /run/secrets
- script consumption
  - scripts can interact directly within /secrets on localhost if only run on localhost
  - but preference should be for /etc/ssl/certs unless logic dictates a more complex dir hierarchy
- for information on how this changes across envs, domains and services, read the private docs

```sh

# copypasta: `cd secrets`
# permissions come from source, not symlinks
# docker secrets bug: @see https://github.com/docker/compose/issues/9648
sudo chown -R $USER:$USER dev.nirv.ai
sudo chown -R $USER:$USER mad.nirv.ai
sudo chown -R consul:consul mesh.nirv.ai

# docker secrets are sourced from /etc/ssl/certs
# thats why we need the symlinks
sudo mkdir /etc/ssl/certs/{dev,mad,mesh}.nirv.ai
sudo mkdir /etc/ssl/certs/{aws,vault}
sudo ln -s `pwd`/dev.nirv.ai/letsencrypt/live/dev.nirv.ai/* /etc/ssl/certs/dev.nirv.ai
sudo ln -s `pwd`/mad.nirv.ai/tls/* /etc/ssl/certs/mad.nirv.ai
sudo ln -s `pwd`/mesh.nirv.ai/tls/* /etc/ssl/certs/mesh.nirv.ai

```
