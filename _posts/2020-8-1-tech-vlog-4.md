---
layout: post
title:  "Tech Vlog #4 - Integrating Docker with GitLab CI"
date:   2020-8-1 17:00
permalink: 'blog/tech-vlog-4'
excerpt: "In this video we'll learn how to integrate Docker into a GitLab CI pipeline, allowing your Rails app to be automatically built and updated in production."
featured-image: "tech-vlog-4.png"
---

<div class="video-container">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/TpVj4oc-3wE" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

Continuing our deep-dive into Docker, this video looks at how we can integrate `Docker` and `Docker Compose` into GitLab CICD (Continuous Integration, Continuous Delivery) pipelines. Could we setup automated builds and deployments so that, with a simple `git push` into `master`, our Rails app is updated all by itself? Let's find out.<br/>

[My DigitalOcean referral link (free $100 credit for you)](https://m.do.co/c/0a355ee4921b)

[🎬Playlist of all VLOG entries](https://www.youtube.com/playlist?list=PLZKJZNiPX65uKeoHLLvi2rh25T9PvtAQc)

<b>Provision a remote Docker host using `docker-machine` and the `generic` driver:
```
docker-machine create --driver generic --generic-ip-address x.x.x.x --generic-ssh-user your-ssh-username --generic-ssh-key ~/.ssh/id_rsa your-remote-machine-name
docker-machine ssh remote-machine-name "sudo apt-get install docker-compose -y"
```

<b>SSL files that must be copied into environment variables and `echo`'d to files within GitLab CI.</b>
```
# Don't forget to replace <remote-machine-name> with the name you gave your remote host in Docker machine.

~/.docker/machine/certs/ca-key.pem
~/.docker/machine/certs/key.pem
~/.docker/machine/machines/<remote-machine-name>/ca.pem
~/.docker/machine/machines/<remote-machine-name>/cert.pem
```

<b>Dockerfile</b>
```
image: docker/compose

stages:
  - build
  - deploy

before_script:
  - mkdir /root/.docker
  - echo "$DOCKER_CA_KEY_PEM" > "/root/.docker/ca-key.pem"
  - echo "$DOCKER_CA_PEM" > "/root/.docker/ca.pem"
  - echo "$DOCKER_CERT_PEM" > "/root/.docker/cert.pem"
  - echo "$DOCKER_KEY_PEM" > "/root/.docker/key.pem"

build:
  stage: build
  only:
    - master

  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --pull --build-arg RAILS_MASTER_KEY=$RAILS_MASTER_KEY --tag $CI_REGISTRY/mindfulchoices/app:latest .
    - docker push $CI_REGISTRY/mindfulchoices/app:latest

deploy:
  stage: deploy
  only:
    - master

  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker-compose -f docker-compose.prod.yml pull
    - docker-compose -f docker-compose.prod.yml run app /bin/bash -c "bundle exec rake db:migrate || bundle exec rake db:setup"
    - docker-compose -f docker-compose.prod.yml stop
    - docker-compose -p 'mindful' -f docker-compose.prod.yml up --detach --remove-orphans

```

<b>docker-compose.prod.yml</b>
```yml
version: '3'

volumes:
  uploaded-resources:
    external: false
  letsencrypt:
    external: false

services:
  # Ruby on Rails application
  app:
    image: registry.gitlab.com/mindfulchoices/app
    depends_on:
      - db
    environment:
      RACK_ENV: production
    volumes:
      - uploaded-resources:/app/public/system
    restart: unless-stopped

  # The postgres docker images will automatically mount their own persistent volume.
  # The name of the volume will be pseudorandom, not friendly. Take care when deleting volumes on the host.
  db:
    image: postgres:latest
    environment:
      - POSTGRES_HOST_AUTH_METHOD=trust
    restart: unless-stopped

  # Our custom docker image containing the NGINX configs for our domains.
  # Warning: The Docker composition will fail to boot if any of the domains referenced in NGINX fail to resolve to the host.
  # This is due to the startup script in staticfloat/nginx-certbot triggering the certbot ACME challenge for all referenced domains.
  # Change `nginx-ssl:latest` to `nginx-ssl:testing` to only generate certs for staging.mindfulchoices.co.uk
  # This allows the production stack to load without having to update the DNS entires for *all* our domains.
  nginx:
    image: registry.gitlab.com/mindfulchoices/nginx-ssl:latest
    volumes:
      - letsencrypt:/etc/letsencrypt
    ports:
      - "80:80"
      - "443:443"
    environment:
      CERTBOT_EMAIL: webmaster@mindfulchoices.co.uk
    depends_on:
      - app
    restart: unless-stopped
```

<b>.gitlab-ci.yml</b>
```yml
image: docker/compose

stages:
  - build
  - deploy

before_script:
  - mkdir /root/.docker
  - echo "$DOCKER_CA_KEY_PEM" > "/root/.docker/ca-key.pem"
  - echo "$DOCKER_CA_PEM" > "/root/.docker/ca.pem"
  - echo "$DOCKER_CERT_PEM" > "/root/.docker/cert.pem"
  - echo "$DOCKER_KEY_PEM" > "/root/.docker/key.pem"

build:
  stage: build
  only:
    - master

  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build --pull --build-arg RAILS_MASTER_KEY=$RAILS_MASTER_KEY --tag $CI_REGISTRY/mindfulchoices/app:latest .
    - docker push $CI_REGISTRY/mindfulchoices/app:latest

deploy:
  stage: deploy
  only:
    - master

  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker-compose -f docker-compose.prod.yml pull
    - docker-compose -f docker-compose.prod.yml run app /bin/bash -c "bundle exec rake db:migrate || bundle exec rake db:setup"
    - docker-compose -f docker-compose.prod.yml stop
    - docker-compose -p 'mindful' -f docker-compose.prod.yml up --detach --remove-orphans
```
