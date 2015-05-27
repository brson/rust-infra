# Links Ahoy

* [rust-buildbot](https://github.com/rust-lang/rust-buildbot). Our configuration for [buildbot](http://buildbot.net/).
* [Our buildbot deployment](http://buildbot.rust-lang.org/)
* [Homu](https://github.com/barosl/homu). Homu is a script we use with buildbot to do ['pre-commit' testing](http://graydon2.dreamwidth.org/1597.html).
* [Our homu deployment](http://buildbot.rust-lang.org/homu/)
* [Homu Rust queue](http://buildbot.rust-lang.org/homu/queue/rust)
* [Homu Cargo queue](http://buildbot.rust-lang.org/homu/queue/cargo)
* [Travis dashboard](http://buildbot.rust-lang.org/travis/travis.html). We use Travis to test rust-lang crates that live outside of rust-lang/rust.
* [Automation metabug](https://github.com/rust-lang/rust/issues/17356). The evergrowing list of things to do.
* [rust-admin](https://github.com/brson/rust-admin). Old information, sometimes still worth looking at. This repo is private because it contains secrets.
* [AWS console](https://console.aws.amazon.com/console/). Login page for AWS.
* [play.rust-lang.org](https://play.rust-lang.org/). Our online Rust evaluator. It has a cross-site API so that it can be used by other projects, e.g. rust-by-example lets all examples be evaluated on play.rlo.
* [rustup.sh](http://github.com/rust-lang/rustup). A script commonly used to download the compiler from release channels. It interprets the metadata uploaded by the distribution builders.
* [rust-packaging](https://github.com/rust-lang/rust-packaging) - The script run by the 'dist-packaging' builders to produce the final Rust builds in a variety of packaging formats.
* [playpen](https://github.com/rust-lang/rust-playpen) - The code that runs play.rust-lang.org.



# SSH and security tips

Learn to love ssh and ssh-agent. Any time you ssh into another machine I suggest using the `-A` flag. This will forward your ssh-agent keyring to the remote machine so you can further authenticate as yourself when ssh'ing into other machines or pushing git repos.

You'll inevitably need access to the Mozilla network from outside the office. You do this by registering your SSH key with LDAP, then ssh'ing to e.g. banderson@ssh.mozilla.com. From there you can get to other machines, including our AWS bastion server (a 'bastion' is a secure server you access the rest of the system through).

I suggest turning on 2-factor authentication on GitHub.

As somebody responsible for Rust infrastructure you are going to have access to a lot of secrets which, if compromised, could result in major damage to the project. Please take steps to secure them. Personally I keep my stuff on a secure [Wuala](https://www.wuala.com) drive then further encrypted with [encfs](https://vgough.github.io/encfs/) (though note that encfs is unmaintained and has known security issues - it is not sufficient on its own and there are probably better solutions now). If you can, don't permanently decrypt the GPG messages you receive containing secrets - just run gpg again every time you need access.

Put a password on your GPG key.




# AWS

Most of our infrastructure is on AWS.

The bastion is at the public IP 54.215.17.149, and called 'bastion' in EC2. All access goes through it. It's used to get to either the production or development master or slaves. It's in the `rust-bastion` security group, which only allows connections from specific IP's (currently only the MV office), so to access the automation from outside of MV the security group must be augmented.

We use EC2, S3 and CloudFront. CloudFront is a content delivery network used soley for serving our s3 bucket over HTTPS via static.rust-lang.org.

## Accessing AWS machines

Access to AWS machines requires the *shared* buildbot-west-slave-key.pem SSH key. Add it to your keyring with `ssh-add buildbot-west-slave-key.pem`, then `ssh -A ubuntu@54.215.17.49` to get into the bastion. From there you can ssh to other machines.

Windows machines on AWS are accessed by tunnelling RDP through the bastion. Establish the tunnel from your local machine by running e.g. `ssh -L54.215.17.49:3389:$windows_slave_address:3389`, then rdp to localhost e.g. `rdesktop localhost`.

## Machines

Descriptions of some AWS machines, as labeled in EC2:

* bastion - The machine you access other AWS machines through
* rust-prod-master - The production buildbot build master, also runs other related scripts.
* rust-dev-master - The development buildbot build master.
* prod-slave-* - Production buildslaves. These start and stop on demand.
* doc.rust-lang.org - Mislabeled. An nginx proxy for various purposes, mostly wrapping HTTP services in HTTPS.
* play.rust-lang.org - The machine hosting play.rust-lang.org. Runs arbitrary Rust code in a sandbox.

## Proxy setup

The machine labeled doc.rust-lang.org is an nginx proxy that we use for a variety of purposes. It's most important function is serving HTTP content over HTTPS.

This machine contains the rust-lang.org wildcard TLS cert.




# Buildbot

We have a decently customized buildbot environment, the [source of which](https://github.com/rust-lang/rust-buildbot) is open. It has been developed incrementally and haphazardly over years. Due to that and the somewhat complex nature of buildbot the program logic can take a while to grasp.

We run buildbot in two environments: 'prod' and 'dev'. The dev environent tends to only get used for developing 'significant' changes to rust-buildbot and it often lags behind the prod environment.

Buildbot is used for three purposes: continuous integration, distribution, try builds (a service for devs to test their changes on our bots).

We further customize buildbot by driving continuous integration through [Homu](https://github.com/rust-lang/rust-buildbot) which implements ['pre-commit' testing](http://graydon2.dreamwidth.org/1597.html), testing GitHub pull requests *before* merging them into master. This is an unorthodox setup - most CI systems merge then test. This setup though guarantees that our build is always green - there are no 'sheriffs' who have to go through the tree after the fact fixing breakage.

There are two accounts used to log into buildbot via the web interface: 'any-build' and 'rust'. The 'rust' user can start and stop builds, but not for any 'dist' builders (the ones that publish releases). 'any-build' has full access. The passwords to these are shared.

## The build master

The buildbot architecture features a single master that coordinates the build and an arbitrary number of slaves.

The production build master is located at the public IP 54.241.248.193 and has the EC2 name, rust-prod-master.

The build master is mostly operated by the 'rustbuild' user. To access, ssh from the bastion to `rustbuild@10.190.147.69` (the private IP).

Don't bother trying to `buildbot restart master` - the slave shutdown takes a long time and the restart will time out. When shutting down with `buildbot stop master`, make sure you wait until buildbot actually exits (it takes a long time to wait on the EC2 slaves).

Buildbot listens *locally* on port 8010 via HTTP, and Homu on port 7942. A logal nginx instance is configured via `/etc/nginx/available-sites/default` to proxy them both to port 80.

The buildbot source is installed at ~/rust-buildbot, homu at ~/homu.

The build master contains an s3cmd configured so that - when necessary - people can upload directly to s3. The automation does all of the interfacing to s3 under normal circumstances.

The rustbuild user runs a number of cronjobs.

The build master contains critical secrets, including the GPG subkey for signing releases and an s3 access token.

Don't try to restart buildbot by running `buildbot restart`. Because EC2 slaves can take a long time to shut down (and sometimes never halt due to buildbot bugs), the restart will almost always timeout. Instead `buildbot stop`, then `ps -A | grep buildbot` until you see the process go away, then `buildbot start` (ikr?). These commands are always in the bash history.

## Important files

* ~/rust-buildbot/master/master.cfg - The buildbot script. It's python - buildbot convention uses the .cfg extension though.
* ~/rust-buildbot/master/master.cfx.txt - Environment specific configuration. Contains secrets.
* ~/rust-buildbot/master/slave-list.txt - The list of slaves. Contains secrets.
* ~/homu/cfg.toml - The homu config
* ~/invalidate.sh - A fairly ineffective script for daily invalidating artifacts on Cloudfront, our CDN.
* ~/travis - The script that creates http://buildbot.rust-lang.org/travis/travis.html periodically via cron job
* ~/update-rust-dist-index.sh - Another script that generates index.html files for our s3 bucket.
* ~/s3-directory-listing - Used by the above.

The other stuff in rustbuild's home directory is (probably) old and unused.

## The build slaves

The build master starts EC2 slaves on demand, calling them `prod-slave-$name`. Buildslaves communicate to the master via stunnel.

When an EC2 slave boots it runs ~/rust-buildbot/setup-slave.sh, which pulls the slave name and password, and the IP address of the build master from EC2 data, configures and runs stunnel and the buildslave.

The macs are configured manually, and should restart on reboot, but in practice they tend to need to be fiddled with after a reboot.

The macs live on brson's desk, at the following addresses:

- rust-mac3.corp.mtv2.mozilla.com
- rust-mac4.corp.mtv2.mozilla.com
- rust-mac5.corp.mtv2.mozilla.com
- rust-mac6.corp.mtv2.mozilla.com

These can only be accessed from within the Mozilla network. To get to them from outside first ssh to ssh.mozilla.com.

## Continuous Integration

Continuous integration is done by the 'auto-*' builders in buildbot, builds on which are started by homu pushing to the 'auto' branch.

Homu is running in an instance of
[screen](https://www.gnu.org/software/screen/) under the *root*
account, not *rustbuild*. To access it ssh from bastion to the build
master as root, then run `screen -R` (reattach to existing session).
To detach press `Ctrl-A`, then `D` (`Ctrl-A` is the screen command key).

Cargo's continuous integration is done by the 'cargo-*' builders, also driven by homu.

TODO. There's probably a lot more to say about Rust continuous integration.

## Distribution

The 'stable-dist-*', 'beta-dist-*', and 'nightly-dist-*' builders are all used for publishing releases of Rust. Their interaction is somewhat complex, customized in master.cfg. These builders upload artifacts to the s3 'dist/' or 'cargo-dit/' which are then interpreted by various tools, like rustup.sh.

TODO. There's probably a lot more to say about Rust distribution.

# Travis

TODO

# crates.io

crates.io runs on Heroku. TODO.

# play.rust-lang.org

TODO

# S3

# DNS addresses

A list of DNS addresses owned by rust-lang and what they are.

* www.rust-lang.org - The website, hosted via GitHub pages.
* static.rust-lang.org - Mostly used for distributing binaries, hosted via CloudFront, maps directly to the static-rust-lang-org s3 bucket.
* dev-static.rust-lang.org - The equivalent of the above in dev.
* doc.rust-lang.org - The doc server, proxies over HTTPS via nginx.
* users.rust-lang.org - A [Discourse](http://discourse.org) instance for Rust users, hosted by Discourse.
* internals.rust-lang.org - A Discourse instance for Rust developers, hosted by Discourse.
* blog.rust-lang.org - Hosted by GitHub pages.
* play.rust-lang.org - Hosted on AWS.
* playtest.rust-lang.org
* {imap,pop,smtp}.rust-lang.org - IMAP acess to @rust-lang emails. Rarely used, though discourse may be using the smtp server.

We use easydns.com. Ask brson if you need access.
