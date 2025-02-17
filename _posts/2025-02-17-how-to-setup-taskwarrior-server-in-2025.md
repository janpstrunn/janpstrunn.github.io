---
title: How To Setup Taskwarrior Server In 2025
date: 2025-02-17
description: Learn how to sync your tasks the right way!
categories:
  - Software
tags:
  - task-management
  - cli
image:
  path: /assets/img/2025-02-17-how-to-setup-taskwarrior-server-in-2025.avif
  alt: How to Setup Taskwarrior Server in 2025
---

# How To Setup Taskwarrior Server In 2025

## Introduction

This is a guide on how to setup a Taskwarrior server for the 2.x.x and 3.x.x versions.

The main differences between the 2.x.x era and the new current 3.x.x versions of taskwarrior is mainly the format where these two are presented and their syncing methods.

- Taskwarrior V2
  - Uses **JSON** as its data files, split in four main files: pending.data, completed.data, backlog.data and undo.data
  - Uses `taskd` for syncing files across devices
- Taskwarrior V3
  - Uses **SQLite** as its data file called: taskchampion-sqlite.db
  - Uses cloud services (e.g. Google Cloud and AWS S3) or taskchampion for syncing

In short, the 3.x.x series have a completed revamp on how syncing works and the way data is stored to improve performance at the very, very long term. I'll be skipping the JSON and SQLite details, but it's useful to know that SQLite is highly more efficient in dealing with high amounts of data (that happens in the long term).

Since the major taskwarrior upgrade, I found out that many projects associated to taskwarrior are no longer maintained and were created for the 2.x.x versions of it. For new users, taskwarrior 3.x.x is the best good to go, but for the users that were already using 2.x.x versions may have created a workflow around some tools like [taskwarrior-flutter](https://github.com/CCExtractor/taskwarrior-flutter) and [WingTask](https://app.wingtask.com/) that allowed taskwarrior to expanded to mobile devices seamlessly without sacrificing any bit of security.

If you never wondered about augmenting your taskwarrior setup, you may want to give at look at [Taskwarrior Tools](https://taskwarrior.org/tools/). It's a very big lists of all available taskwarrior related projects that can do all kinds of stuff.

For this guide, I will be focusing in two amazing tools for running a server:

- [taskserver](https://github.com/GothenburgBitFactory/taskserver) for the 2.x.x series
- [taskchampion](https://github.com/GothenburgBitFactory/taskchampion-sync-server) for the 3.x.x series.

> [!NOTE]
> I'll not be talking about how to setup Google Cloud or AWS S3 for Takswarrior V3 in this post

## Taskwarrior V2

This will be a bare, simple, straight-forward, poor and dry redo of the already existing guide on how to setup a taskserver: Available at: [taskserver-setup](https://gothenburgbitfactory.org/taskserver-setup/).

If you get questions on the setup, I do recommend you check the official guide, since it does give better explanations.

> **Warning:** Taskserver is only compatible with Taskwarrior 2.x.x, and is no longer actively developed.
> {: .prompt-warning}

### Dependencies

```
GnuTLS (ideally version 3.2 or newer)
libuuid
cmake 2.8 or newer
make
gcc 4.7 or clang 3.0 or newer
```

### Setup

#### Installation

Download the official Github repository

```bash
git clone --recurse-submodules=yes \
      https://github.com/GothenburgBitFactory/taskserver.git \
      taskserver.git
cd taskserver.git
```

Change branch to `master` to get stable version

```bash
git checkout master
```

Init submodules

```bash
git submodule init
```

Build your taskserver

```bash
cmake -DCMAKE_BUILD_TYPE=release .
make
sudo make install
```

Verify your installation

```bash
taskd -v
```

#### Config

Taskserver data relies on an environment variable: $TASKDDATA (Has two 'D's). To set this up, add the following line to your `.shellrc` or `.env` and source it.

```bash
export TASKDDATA=/var/taskd
```

In the Github repository you just cloned, there will be a directory called `pki`. Move it to $TASKDDATA.

```bash
mv pki $TASKDDATA
```

The following command will do at lot. It will move your location to $TASKDDATA/pki and setup the certificates by running one of many scripts, and copy them to the $TASKDDATA directory.

```bash
cd $TASKDDATA/pki
./generate
cp client.cert.pem $TASKDDATA
cp client.key.pem $TASKDDATA
cp server.cert.pem $TASKDDATA
cp server.key.pem $TASKDDATA
cp server.crl.pem $TASKDDATA
cp ca.cert.pem $TASKDDATA
```

Now you will have to tell taskserver to read these certificates, by running:

```bash
taskd config --force client.cert $TASKDDATA/client.cert.pem
taskd config --force client.key $TASKDDATA/client.key.pem
taskd config --force server.cert $TASKDDATA/server.cert.pem
taskd config --force server.key $TASKDDATA/server.key.pem
taskd config --force server.crl $TASKDDATA/server.crl.pem
taskd config --force ca.cert $TASKDDATA/ca.cert.pem
```

Next step is to setup some other configurations: Set a log and pid file, and set the address to listen to:

```bash
cd $TASKDDATA/..
taskd config --force log $PWD/taskd.log
taskd config --force pid.file $PWD/taskd.pid
taskd config --force server address:53589
```

Launching the server is already possible using the `taskdctl start` command, but the configuration is not done yet. To let systemd start this automatically for you, you may set a service. **This step is optional**:

```systemd
[Unit]
Description=Secure server providing multi-user, multi-client access to Taskwarrior data
Requires=network.target
After=network.target
Documentation=http://taskwarrior.org/docs/#taskd

[Service]
ExecStart=/usr/local/bin/taskd server --data $TASKDDATA
Type=simple
User=$TASKDUSER
Group=$TASKDGROUP
WorkingDirectory=$TASKDDATA
PrivateTmp=true
InaccessibleDirectories=/home /root /boot /opt /mnt /media
ReadOnlyDirectories=/etc /usr

[Install]
WantedBy=multi-user.target
```

After creating this file (e.g. taskd.service), you may start it as follows:

```bash
cp taskd.service /etc/systemd/system
systemctl daemon-reload
systemctl start taskd.service
systemctl status taskd.service
```

If you do not get any error, you may now enable the service running `systemctl enable taskd.service`.

Now, the final part of the configuration comes. We will need to configure an organization and user and set their certificates as well. The following lines can be adjusted to your preferences.

```bash
taskd add org 'Public' # Creates the Public organization
taskd add user 'Public' 'Username' # Creates an user to an organization
```

Now to setup the user certificates you will have to go back to `pki` directory and run the `generate.client` script.

> [!NOTE]
> The `generate.client` requires some of the certificates you generated on the server setup to work. In case you moved it from there, make a copy to the `pki` directory

```bash
cd ~/$TASKDDATA$/pki
./generate.client Username
cp username.cert.pem ~/.task
cp username.key.pem ~/.task
cp ca.cert.pem ~/.task
```

And finally, we have to tell taskwarrior to read these files and know where the server is located at.

- You may switch `host.domain` to `address`, where it can be a public or local IP Address
  - If you want to run the server locally, and not inside a VM or Cloud Services, you may set a static IP Address so you configuration won't break
- Change the credentials to the one `generate.client` created for you. If you forgot, it's the user directory at $TASKDDATA/org/user/

```bash
task config taskd.certificate -- ~/.task/first_last.cert.pem
task config taskd.key -- ~/.task/first_last.key.pem
task config taskd.ca -- ~/.task/ca.cert.pem
task config taskd.server -- host.domain:53589
task config taskd.credentials -- Public/Username/cf31f287-ee9e-43a8-843e-e8bbd5de4294
```

### Usage

You must send all your tasks to the server running:

> [!CAUTION]
> This is something you should only do once on only one device.

```bash
task sync init
```

The command you will be most using is `task sync` for everytime you want to sync your tasks with the server. Optionally, to spare the necessity to run this every single time, you may add a `crontab` to run this command every 10 minutes, exemplified as follows:

```cron
*/10 * * * * /usr/bin/task sync >/dev/null 2>&1
```

That's it for Taskwarrior V2, enjoy you syncing across devices! Just remember to export the same configuration lines from your `.taskrc` to other devices so they will listen to the same place.

## Taskwarrior V3

If you read the Taskwarrior V2 setup, fear not, the V3 series is much easier to setup. The official step-by-step is available at [taskchampion-sync-server](https://github.com/GothenburgBitFactory/taskchampion-sync-server?tab=readme-ov-file#building)

### Dependencies

```bash
rustup or cargo
```

### Setup

#### Installation

Install `rustup` and set branch to stable

```bash
sudo pacman -S rustup # For Arch Linux Distros
rustup default stable
```

Clone the official Github repository and build `taskchampion-sync-server` to the release version

```bash
git clone https://github.com/GothenburgBitFactory/taskchampion-sync-server.git
cd taskchampion-sync-server
cargo build --release
```

The binary will be created at `/target/release/taskchampion-sync-server`, and you may move it to any directory configured in $PATH environment variable. If you do not have a $PATH setup, you may add the next line to your `.shellrc` or `.env` and source it. It will make your `~/local/bin/` to listen to binaries allowing easier execution by the command line.

```bash
export PATH="$HOME/.local/bin/:$PATH"
```

You can already run you server with the following command `taskchampion-sync-server`. But you may set some options to it. We will be using the `-C` and `-d` options.

To use `-C` or `--allow-client-id`, you may create an UUID running the `uuidgen` command (mostly available at Linux systems).
To use `-d` or `--data-dir`, you have to point the path to where taskchampion should save the files.
If your server is not pointing to the `8080` port, you must set this, if not it will not work. You may do it using `-p=8080`.

> [!WARNING]
> If you already use taskwarrior, do not point `-d` to the existing directory, because it will wipe all your data!

To set this up, follow the example:

```bash
taskchampion-sync-server -C 6b00cabe-4e17-405f-878b-5b61a99cf326 -d /home/user/.taskchampion
```

Finally, you will have to tell taskwarrior where your server is, and how it should talk to it. So you must provide three configuration details to it.

- **Server URL:** Must include the scheme, such as 'http://' or 'https://'.
  - Example: `http://localhost:8080` or `http://192.168.1.43` (for a static IP address)
- **Client ID:** Is the UUID you have generated previously using `uuidgen`
- **Encryption Secret:** This can be **any secret string**, and must match for all replicas sharing tasks. You may generate it using `pwgen`

```bash
task config sync.server.url               <url>
task config sync.server.client_id         <client_id>
task config sync.encryption_secret        <encryption_secret>
```

And of course, to ease up the process to run the server, you may configure a service to systemd. You may find the following example useful. You may set the environment variables or replace the lines.

```systemd
[Unit]
Description=TaskChampion is the task database Taskwarrior uses to store and sync tasks
Requires=network.target
After=network.target
Documentation=https://github.com/GothenburgBitFactory/taskchampion-sync-server

[Service]
ExecStart=/usr/local/bin/taskchampion-sync-server -C $TASKCHAMPUUID -d $TASKCHAMPDATA
Type=simple
User=$TASKDUSER
Group=$TASKDGROUP
WorkingDirectory=$TASKCHAMPDATA
PrivateTmp=true
InaccessibleDirectories=/home /root /boot /opt /mnt /media
ReadOnlyDirectories=/etc /usr

[Install]
WantedBy=multi-user.target
```

### Usage

You must send all your tasks to the server running:

> [!CAUTION]
> This is something you should only do once on only one device.

```bash
task sync init
```

The command you will be most using is `task sync` for everytime you want to sync your tasks with the server. Optionally, to spare the necessity to run this every single time, you may add a `crontab` to run this command every 10 minutes, exemplified as follows:

```cron
*/10 * * * * /usr/bin/task sync >/dev/null 2>&1
```

That's it for Taskwarrior V3, enjoy you syncing across devices! Just remember to export the same configuration lines from your `.taskrc` to other devices so they will listen to the same place.

## Finals

No matter if you use Taskwarrior V2 and V3 you can now sync your data across different devices as you wish the official way!

Before I could ever knew about it, I used to use Syncthing for syncing my tasks, and they were always having merge conflicts, so I had to try something else. And running a taskwarrior server is definitively a better option.

Note that both Taskwarrior V2 and V3 can export your data in JSON format, which means you can import your data from V2 to V3 or vice-versa, but know that doing it the other way is not officially praised, however it works fine.
