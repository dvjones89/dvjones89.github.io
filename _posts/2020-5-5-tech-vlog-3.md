---
layout: post
title:  "Tech Vlog #3 - Dockerising Rails, Postgres and NGINX."
date:   2020-5-5 12:00
permalink: 'blog/tech-vlog-3'
excerpt: "In this video we'll learn how to \"Dockerise\" a Ruby on Rails application, spin up a Postgres database and wire up an NGINX web server, all using the magic of Docker and Docker Compose."
featured-image: "tech-vlog-3.jpg"
---

<div class="video-container">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/BLFZ9KJLoaY" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

Whilst waiting on the delivery of some Arduino components, I decided to take a closer look at Docker 🐳. In this video we'll learn how to "Dockerise" a Ruby on Rails application, spin up a Postgres database and wire up an NGINX web server, all using the magic of Docker and Docker Compose. <br/> 

[My DigitalOcean referral link (free $100 credit for you)](https://m.do.co/c/0a355ee4921b)

[🎬Playlist of all Tech VLOG entries](https://www.youtube.com/playlist?list=PLZKJZNiPX65uKeoHLLvi2rh25T9PvtAQc)

<b>Dockerfile</b>
```
FROM ruby:2.6.5

# Create a home directory where the magic happens
ENV APP_HOME /app
RUN mkdir $APP_HOME
WORKDIR $APP_HOME

# Install Chromium for Capybara feature specs
RUN wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN sh -c 'echo "deb https://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list'
RUN apt-get update
RUN apt-get install -y google-chrome-stable

# Install our gem dependencies
ADD Gemfile* $APP_HOME/
RUN bundle install

# Load all of our source code into the container
ADD . $APP_HOME

# Compile assets
RUN bundle exec rake assets:precompile

# Start the web server
EXPOSE 3000
RUN rm /app/tmp/pids/server.pid
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]

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
      RAILS_ENV: production
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

  # https://github.com/staticfloat/docker-nginx-certbot
  nginx:
    image: staticfloat/nginx-certbot
    volumes:
      - /etc/nginx/user.conf.d:/etc/nginx/user.conf.d:ro
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
