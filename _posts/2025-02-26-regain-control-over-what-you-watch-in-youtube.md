---
title: Regain Control Over What You Watch in YouTube
date: 2025-02-26
description: Make yourself a step from being lured by useless recommendations in YouTube with Invidious
categories:
  - Software
tags:
  - cli
  - self-host
image:
  path: /assets/img/2025-02-26-regain-control-over-what-you-watch-in-youtube.avif
  alt: Regain control now!
---

# Regain Control Over What You Watch in YouTube

## Introduction

In this post I'll show how to self-host Invidious with Docker, and configure FreeTube to use your Invidious Instance for best performance, enjoy ad-free videos and regain control over what you watch. While self-hosting Invidious is not mandatory, it allows you to watch YouTube videos without relying on the YouTube API to improve your privacy, and it does also give you some unique perks like accounts (but I'll not cover this).

YouTube is inevitable. If you want to watch anything, is it a cook recipe guide, hear some music, learn how to do something, get entertained or anything else you may think, you may find yourself in YouTube. But there is a catch. YouTube has over **TOO MUCH** features, and mostly of them do not make everyone very happy, and some of them may put you in a vicious loop, seeking for cheap dopamine.

I used to use YouTube like any other regular person, until I stepped to the world of the privacy concerned, and I found a YouTube Frontend called CloudTube or Cadence, and in the very homepage, that was very clean, without a single recommendation, trends or anything else, other was a link called: [...can't think of anything?](https://tube.cadence.moe/cant-think). This link let me to a page with some kind of story and an audio. In few words, what those words meant was, that you may find yourself getting to YouTube with no goal, and wait for something wonderful to happen, just like finding an amazing video that may change your life, when there was no reason to you be there in the first place.

Since I read that post, I've started to get more aware of what I was consuming online, not only for YouTube. And so I wanted to keep all the content I consume aligned to what was my purpose, what I actually wanted to learn, and what I wanted to do.

Them I found an amazing application called [FreeTube](https://freetubeapp.io/). This was a much better way to watch YouTube videos in an interface that was highly customizable, and the options to disable some content from showing up such as Trending, Popular Videos and Shorts. And them that was my choice. I completely shredded my YouTube data and moved to FreeTube ways of dealing with history, playlist and subscriptions.

By default, FreeTube uses its API to load videos and play them straight from YouTube API, and remove any kind of ads from it. From my experience, everytime youtube.js had changes, FreeTube API was directly affected, and I had a bad time until it's fixed, until a time I decided to circumvent this issue myself.

FreeTube has the option to use an Invidious Instance as its backend, which does not rely on the youtube.js to work, and strip the ads itself. But Invidious providers doesn't offer very highly performance compared to FreeTube API. That was when I tried to self-host Invidious.

## Setup

### Dependencies

- docker and docker-compose, or podman and podman-compose
- git
- openssl (optional)

> I do use podman for containerization, and all my scripts shown in this post also use podman. You may tweak them if you want to use docker instead
{: .prompt-info }
### Installation

The official documentation may be found at [Invidious/Installation](https://docs.invidious.io/installation/#docker-compose-method-production)

First you will need to clone the official Invidious repository to get all the necessary files for Invidious to work.

> If you try to run docker-compose just after cloning the Invidious repository, you will use the development configuration, and not the production configuration
{: .prompt-warning }

```bash
git clone https://github.com/iv-org/invidious.git
cd invidious
```

There is only one file you must change in this repository: docker-compose.yml

You should replace the docker-compose.yml in the Invidious repository with the following:

```yml
version: "3"
services:

  invidious:
    image: quay.io/invidious/invidious:latest
    # image: quay.io/invidious/invidious:latest-arm64 # ARM64/AArch64 devices
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      # Please read the following file for a comprehensive list of all available
      # configuration options and their associated syntax:
      # https://github.com/iv-org/invidious/blob/master/config/config.example.yml
      INVIDIOUS_CONFIG: |
        db:
          dbname: invidious
          user: kemal
          password: kemal
          host: invidious-db
          port: 5432
        check_tables: true
        signature_server: inv_sig_helper:12999
        visitor_data: CHANGE_ME
        po_token: CHANGE_ME
        # external_port:
        # domain:
        # https_only: false
        # statistics_enabled: false
        hmac_key: "CHANGE_ME!!"
    healthcheck:
      test: wget -nv --tries=1 --spider http://127.0.0.1:3000/api/v1/trending || exit 1
      interval: 30s
      timeout: 5s
      retries: 2
    logging:
      options:
        max-size: "1G"
        max-file: "4"
    depends_on:
      - invidious-db

  inv_sig_helper:
    image: quay.io/invidious/inv-sig-helper:latest
    init: true
    command: ["--tcp", "0.0.0.0:12999"]
    environment:
      - RUST_LOG=info
    restart: unless-stopped
    cap_drop:
      - ALL
    read_only: true
    security_opt:
      - no-new-privileges:true

  invidious-db:
    image: docker.io/library/postgres:14
    restart: unless-stopped
    volumes:
      - postgresdata:/var/lib/postgresql/data
      - ./config/sql:/config/sql
      - ./docker/init-invidious-db.sh:/docker-entrypoint-initdb.d/init-invidious-db.sh
    environment:
      POSTGRES_DB: invidious
      POSTGRES_USER: kemal
      POSTGRES_PASSWORD: kemal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]

volumes:
  postgresdata:
```

But it's not over yet. We need to change a few things in this configuration, such as the *visitor_data*, *po_token* and the *hmac_key*.

To better explain what are the *visitor_data* and *po_token*, they allow to bypass verification checks from YouTube, however as it is an unique identifier, it may be used as a fingerprinting technique to trace you. To work around this the values may be changed from time to time. While there is no official tool to do this, you may use a shell script I've developed to ensure you harden your privacy, which I'll show in Usage section.

Do get these values manually you will need to run another docker:

```bash
docker run quay.io/invidious/youtube-trusted-session-generator
```

At the end of the executing, it will return the *visitor_data* and *po_token* you will need to add to your docker-compose.yml file.

The script I've made automatically runs this docker, gets these values and overwrite the current ones in the docker.compose.yml file. The script is shown down below:

> You may change the invidious_config path to your actual invidious directory.
{: .prompt-warning }

```bash
#!/bin/env bash

invidious_config=$HOME/invidious/docker-compose.yml

docker run quay.io/invidious/youtube-trusted-session-generator >token.txt

visitor_data=$(grep "visitor_data" token.txt | awk -F ':' '{print $2}')
po_token=$(grep "po_token" token.txt | awk -F ':' '{print $2}')

sed -i "s/^\([[:space:]]*visitor_data:\)[[:space:]]*.*/\1 $visitor_data/" "$invidious_config" && visitor_ok=True
sed -i "s/^\([[:space:]]*po_token:\)[[:space:]]*.*/\1 $po_token/" "$invidious_config" && po_ok=True

if [ "$visitor_ok" == "True" ] || [ "$po_ok" == "True" ]; then
  echo "Tokens successfully updated!"
else
  echo "Tokens failed to be updated!"
fi
```

The last thing we need to do is to create an *hmac_key*, which it can be any value created by the user. To easily create a *hmac_key* you can run the following command

```bash
openssl rand -hex 20
```

Now that you installed Invidious, you will have to configure FreeTube to listen to you server. It's as easy as to go to the Settings > General > Current Invidious Instance. In the box you will insert the address for the Invidious Instance. If you have not made any changes to the address in the docker-compose.yml file, it'll be http://localhost:3000 by default.

That's it!
## Usage

Now that you got Invidious as you backend, and using FreeTube as your frontend. You can now explore all YouTube videos without any kind of ads, and in a very customizable environment.

I've made a script (shown down below) that I always use to open my FreeTube application. It will start the container if it's not running already, but if it does I'll skip the boot and will launch the FreeTube application. Note that the script sleeps for 5 seconds when a container is started before it launches the FreeTube application. This is to give time to the container to be fully operational, else the application will fail to use the self-hosted Invidious instance and will fall back to the FreeTube's API.

It does also ensure your tokens are refreshed once per session. The official documentation mention some sort of "change every X hours". But I noticed that generating tokens often may produce broken tokens that may not work as expected, that's why I kept it once per session. The script can be found here:

```bash
#!/bin/env bash

invidious_config=$HOME/invidious/

function startup() {
  if podman ps | grep -q "invidious"; then
    echo "Invidious Instance is already running."
  else
    cd "$invidious_config" || {
      notify-send -u critical "Invidious" "An error occurred"
      exit 1
    }
    sh "$HOME/scripts/__invidious-token-generator.sh"
    podman-compose up -d
    echo "Invidious Instance launched."
    sleep 5
  fi
}

function main() {
  startup
  freetube # FreeTube binary
}

main
```

## Conclusion

Why did I do all this hard work just to watch YouTube videos? That's just because Invidious offer more stability rather than FreeTube API, but FreeTube interface is completely superior to Invidious Web Interface, which you can customize it the way you want, is it colors or what sections are visible while navigating through the videos.

I'd say that this is a very cozy way, yet technical to setup FreeTube. While self-hosting Invidious is not a must, you may use the FreeTube API without any problems, but notice that it may come with some issues from time to time.

FreeTube is my solution to watch the videos **I want to watch** rather than what is recommended to me to watch based on algorithms. This save me uncountable amounts of hours from time wasting on videos I wasn't meant to be watching, and keeping the overall experience much more stress-free, by bypassing all ads.

Not only that, but I can have all my data saved locally and exported across different services and clients at any time.

For a last thing, almost off-topic. You can both save your audios or videos from FreeTube or Invidious Web Interface at high-speeds if you self-host an Invidious Instance. But if you want to download videos or audios in a more scriptable way, you may find some of my other scripts interesting, such as: [download-youtube.sh](https://github.com/janpstrunn/dotfiles/blob/main/scripts/__download-youtube.sh) and [download-freetube.sh](https://github.com/janpstrunn/dotfiles/blob/main/scripts/download-freetube.sh). The first one allows you to download any YouTube audio or video from a URL, and even in bulk. The second one allows you to download your FreeTube's Playlists exported to a `.db` format, but it depends on the first one to work.