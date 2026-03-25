---
layout: post
title:  "Installing multiple Ruby versions on Windows"
date:   2026-03-25 22:13:57 +0900
categories: build environment
comments: true
---
While preparing this blog, I referred official guidelines on [GitHub Pages](https://docs.github.com/ko/pages) mostly. Soon, I found it using [Ruby](https://www.ruby-lang.org/), and reminded of the trouble when I managed [Redmine](https://www.redmine.org/).

As I expected, the first thing I determined is choosing which version of Ruby should be installed. Because I want to test my posting first via local server, I need to make very similar environment to GitHub server on my local system. Of course GitHub provides its [dependency versions](https://pages.github.com/versions.json), things are not that simple.

According to version information, I need to install Ruby 3.3.4, but I got several build/install errors during `gem install jekyll -v 3.10.0` or else. After a few version trial, I found 3.1.x is the latest one I can use for my work. But I also want to install the latest version, so I dig more to install multiple versions together.

The following is the process of installing Ruby 3.1 & 4.0 together and setting up for my GitHub Pages project.

## Install Ruby

I love package management system, like apt/dpkg in Debian, port system in Gentoo, or even source based management in *BSD. The point is that I can install/remove/update applications/packages in unified manner.

In Windows world, there was no such centralized package system for a long time. But a few years ago, several attempts are made and the most popular one was [chocolatey](https://chocolatey.org/). I used it for a long time, but in Windows 11, `winget` was introduced and now it is part of Windows Operation System. Now I am using `winget` for managing most of my applications, and I will use it for installing Ruby, too.

Lots of Ruby versions can be found in winget repository, as follows.

| Name | Device ID | Version |
| ---- | --------- | ------- |
| Ruby 3.0 | RubyInstallerTeam.Ruby.3.0 |  3.0.7-1 |
| Ruby 3.1 | RubyInstallerTeam.Ruby.3.1 |  3.1.7-1 |
| Ruby 3.2 | RubyInstallerTeam.Ruby.3.2 | 3.2.10-1 |
| Ruby 3.3 | RubyInstallerTeam.Ruby.3.3 | 3.3.10-1 |
| Ruby 3.4 | RubyInstallerTeam.Ruby.3.4 |  3.4.9-1 |
| Ruby 4.0 | RubyInstallerTeam.Ruby.4.0 |  4.0.2-1 |
| Ruby 3.0 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.3.0 |  3.0.7-1 |
| Ruby 3.1 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.3.1 |  3.1.7-1 |
| Ruby 3.2 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.3.2 | 3.2.10-1 |
| Ruby 3.3 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.3.3 | 3.3.10-1 |
| Ruby 3.4 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.3.4 |  3.4.9-1 |
| Ruby 4.0 with MSYS2 | RubyInstallerTeam.RubyWithDevKit.4.0 |  4.0.2-1 |

Because [jekyll](https://jekyllrb.com/) requires C extension build, I need MSYS2-included versions.

*[NOTE] Maybe I can share MSYS2 between Ruby versions, and MSYS2 can be independantly installed with winget, just installing Ruby may be enough. But a lot of files includeing C headers, libraries, etc. are used for building C extension, and they should be shared across Ruby versions, it is very hard to maintain proper build environment all across versions.*

I installed two versions as following commands:

    winget install RubyInstallerTeam.RubyWithDevKit.3.1 -i
    winget install RubyInstallerTeam.RubyWithDevKit.3.1 -i

Normally, winget installs applications in silent mode, and no user interaction is required. But for Ruby installation, I need to change some install options, so I used `-i` for those purpose. Because I want to install multiple versions of Ruby, I will not add each Ruby executable path to PATH variable and not register specific Ruby binary to `*.rb` extension. **Uncheck these two options** in RubyInstaller and proceed installing.

At the final stage of installation, MSYS2 & MinGW toolchain install will be done, which can be done in command line by `ridk install`. Select default option (`1,3`) to install MSYS2 base systems & MinGW toolchains.

<span style="color:red">**[NB] Don't select option 2 to update MSYS2 system during Ruby 3.1 install. It will update toolchains and may cause build failure in jekyll install.**</span>

Even if I omit adding PATH variable, I can still access each version of ruby executable in "Ruby Command Prompt". If more sofisticate configuration is required, you may need to use proper "Ruby Manager" like [rbenv](https://github.com/RubyMetric/rbenv-for-windows), [RVM](https://github.com/magynhard/rvm-windows#readme), or [uru](https://bitbucket.org/jonforums/uru).

## Install bundler & jekyll

Now installing `bundler` and `jekyll` for each ruby version. Normally the following commands are OK for latest Ruby, but I tried to stick `jekyll` version matched to GitHub. **Start "Ruby Command Prompt" in administative privilege** and use the below commands.

    gem install bundler
    gem install jekyll

For Ruby 3.1,

    gem install bundler
    gem install jekyll -v 3.10.0

***[NOTE] If not using priviledge console, package executables will be installed in `C:\Users\username\AppData\Local\Microsoft\WindowsApps` folder. This is not preferable when installing multiple version of Ruby together. In the other hand, if you install packages in admin mode, executables will be installed on each ruby version installation directory.***

OK, now I am ready for creating new jekyll site.

## Create & Build new jekyll pages

Now I can make a new empty folder for jekyll site, or move to checkouted github folder to create and build pages.

In privileged "Ruby Console", To create jekyll site(`--force` option may be needed if folder already has files inside.):

    jekyll new --skip-bundle [--force] .

After modifying files to fix my purpose, To build site:

    bundle install
    bundle exec jekyll build

To test site locally:

    bundle exec jekyll serve

If want to test in "production" environment (Notice on no space between production and &&):

    set JEKYLL_ENV=production&& bundle exec jekyll serve

*[NOTE] I can do most of works in user-mode console, except `bundle install` command. But for the same reason for `gem` command, `bundle` command should be run in admin-mode to avoid binary version conflict.*