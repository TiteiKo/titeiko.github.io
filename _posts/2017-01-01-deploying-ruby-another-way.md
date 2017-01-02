---
layout: post
title: Deploying Ruby Another Way
image: /images/deploying-ruby-another-way.jpg
date: 2017-01-01T00:00:00.000Z
---

Most of the people I know that work with Ruby deploy in two different ways: capistrano or heroku. It's easy and fast to setup, and it does the job.

But they don't fit my needs (that's not the topic of this article so I'm not going to talk about that), and I had to look for something else.

The retained solution was to build a `.deb` file, as the servers are running on Ubuntu 16.04.

--------------------------------------------------------------------------------

## Basic structure of the package

Diving into deb packaging can be scary. But if you stick to the basics, you can make it way simpler. I made the choice to only generate an installable deb, not a sources one.

```
.
└── DEBIAN
    └── control
```

The `control` file is the most important file to build a package. And to be fair, it's the only one that is **not** specific to the project in its naming and presence.

```
Package: our-project
Version: 2016.12.30-1404_431fc2e
Architecture: amd64
Depends: ruby2.3, libpq-dev
Maintainer: You <you@your-society.com>
Description: This is an awesome project!
```

### Package

This one is easy. It's the name of your package. So choose something meaningful, that does not already exists _(don't name it `nginx` for example...)_.

[See the documentation](https://www.debian.org/doc/debian-policy/ch-controlfields.html#s-f-Source) for restrictions on the name.

### Versioning

The first obstacle when building a package is to choose a versioning system. I love [semver](http://semver.org/) for gems and libraries. But I find it's not adequate for continuous deployment.

**Needs:**

- version have to include the commit hash
- versions have to be incremental, so using only the commit hash is not possible.
- it must be readable, so no `20150080072500` à la capistrano

**Solution:**

`year.month.day-hourminute_hash`

This gives versions like the following: `2016.12.30-1404-431fc2e`

The moment is the one of the build. So you know it has been built on the 30th of december 2016, at 2pm (and 4 minutes). If you need to see what it looked like back then, just checkout the commit to see.

### Dependencies

The `Depends` keyword is followed by a list of dependencies. Those will be installed alongside your package.

In this example, running `apt-get install our-project` will then also install `ruby2.3` _(and its dependencies)_, and `libpq-dev` _(and its dependencies)_

### Architecture

I'll let the [documentation speek for itself](https://www.debian.org/doc/debian-policy/ch-customized-programs.html#s-arch-spec) on this one.

--------------------------------------------------------------------------------

## Adding the project files

Choose where on the filesystem you want your project files to be. For us, it was `/usr/lib/our-project`. And then just go ahead and copy your files.

```
.
├── DEBIAN
│   └── control
└── usr
    └── lib
        └── our-project
            ├── all
            ├── the
            ├── project
            └── files
```

### To add or not to add

I've made the choice to not add the feature and test files to the package. I've also not added files like `.env.development`, `.rubocop.yml` and other development specific files that are unnecessary for the project to run on the servers.

--------------------------------------------------------------------------------

## Adding a service

Ubuntu 16.04 uses `systemd` to handle services (daemons that runs, like your database, webserver, and other stuffs like that).

Making `initd` scripts for ruby applications was _(IMHO)_ a real pain. Systemd has a simpler syntax. For example, using [puma](http://puma.io/) without downtime, your service file could look like this:

```
[Unit]
Description=Run the project web server
Wants=network-online.target
After=network-online.target

[Service]
Type=forking
Restart=always
WorkingDirectory=/usr/lib/our-project/

ExecStart=/usr/local/bin/bundle exec puma --workers 4 --daemon --pidfile /var/run/our-project-web.pid
ExecStop=/usr/local/bin/bundle exec pumactl --pidfile /var/run/our-project-web.pid stop
ExecReload=/usr/local/bin/bundle exec pumactl --pidfile /var/run/our-project-web.pid phased-restart

EnvironmentFile=/usr/lib/our-project/service.env
PIDFile=/var/run/our-project-web.pid

[Install]
WantedBy = multi-user.target
```

Let's go ahead and add this to our directory:

```
.
├── DEBIAN
│   └── control
├── etc
│   └── systemd
│       └── system
│           └── our-project-web.service
└── usr
    └── lib
        └── our-project
            ├── all
            ├── the
            ├── project
            └── files
```

--------------------------------------------------------------------------------

## Bundle hassle

I have tried to prepackage the gems, using

```
bundle install --deployment --without development test
```

But the `pg` gem was reluctant, probably because I was building on a docker container instead of a real server. Anyway, I haven't been able to **really** package the gems, and lost quite some time on that, so I decided to go to the second best thing.

```
bundle package --all
```

If you don't know about the `package` command of Bundler, let's just say that it will take all the gems of your Gemfile and put them **uninstalled** in a `vendor/cache` folder.

Then, during your installation, you'll just have to run the following command to install the gems, without having to download them.

```
bundle install --local --without development test --deployment
```

Doing this avoids 3 major problems:

- installing (especially for the first time) gems without the cached gems means that each of your servers will fetch the gems from rubygems and your others sources. This takes _(a long)_ time, and some bandwidth.
- the `--all` option means that for the gems located on a `git` repository (`gem 'foo', git: '..'`), it will download the code. If you install the packet at a time T on server A, then at time T+1 there's a (breaking) change (like dropping references) on the repository and finally you install the packet at T+2 on server B, server A and B gem code will be the same.
- no problem if one of your sources server is down (which has happened to me more times than I care for)

--------------------------------------------------------------------------------

## postinst

Packages installation are automatic. You don't really have a hand to how the system will copy your files, how it will handle the previous files (if you're updating). So how can you ask your server to install your gems?

```
.
├── DEBIAN
│   ├── control
│   └── postinst
├── etc
│   └── systemd
│       └── system
│           └── our-project-web.service
└── usr
    └── lib
        └── our-project
            ├── all
            ├── the
            ├── project
            └── files
```

The `postinst` file is one of the special debian files. It's a script that will be executed post installation _(as I hope you've guessed looking at the name)_. It has to be executable _(so don't forget to `chmod +x` it)_

It can be any script. The shebang defining the language.

```
#!/usr/bin/env bash

# Install bundler if it's not present
gem list bundler -i || gem install bundler

# Install the packaged gems
cd /usr/lib/our-project
bundle install --deployment --without development test --local

# Reload the service
systemctl daemon-reload
service our-project-web reload
```

--------------------------------------------------------------------------------

## Wrapping up

### Final touch

There you go, you have your folder, ready to be packaged to a nice `.deb`!

```
dpkg-deb --build the_folder our-project_2016.12.30-1404-431fc2e.deb
```

### Pros

- Installing and updating on a server becomes as easy as running a `apt-get install our-project`
- Ruby and all the other dependencies are installed alongside the project.
- Deploying or updating a server doesn't require to have ruby, git or anything installed except ssh.
- You can go crazy and make a core package containing the source code, a package containing the web service and one containing a workers service.
- No more PR to make to add, remove or change ips of servers in your code.

### Cons

- It requires some work at the beginning
- This method is not really accessible, documentation is difficult to find and understand
- If your project is private, you'll need a private apt repository (ex: [aptly](https://www.aptly.info/), [reprepro](https://mirrorer.alioth.debian.org/))
- The `deb` package will only be compatible with debian based distributions
