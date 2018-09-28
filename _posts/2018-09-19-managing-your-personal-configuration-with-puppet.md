---
layout: post
title: "Managing Your Personal Configuration With Puppet"
date: 2018-09-19
tags: [puppet, tools]
---

Managing my configuration has been one of my biggest pain points as a software developer. I've had to set up
countless workstations and laptops, and have dealt with a constant barrage of issues that arise from not having a high
quality configuration management tool.

<!--more-->

Problems:

* Knowledge loss - How did I fix this on other machines? What packages make my tools work again?
* Time waste - If setting up a workstation becomes an all day activity, plus dealing with any residual issues, how much time is wasted?
* Less customized - When our configuration tools are bad, we tend to favor customizing our workstations less, knowing that it will be time consuming and difficult to reproduce later.


For a long time, my naive approach was to have a script that would install a bunch of packages, and then share my dotfiles using a GitHub repo. The script would symlink my dotfiles to the appropriate places, and the script would be run once when I set up a new laptop. This came with its own share of problems. Writing an idempotent bash script that installs packages and manages your workstation can be very difficult and error prone. So once I've bootstrapped the machine with my setup script, I'm back in no mans land, with no good place to put any new configuration/setup options that will be useful any time soon.

I've landed on a solution, using [Puppet](https://puppet.com/) that I think is sustainable, and has empowered me to really customize my workstations without worrying. Before getting into the specifics on my setup, I'll summarize the benefits of using this pattern. Keep in mind that there are other tools, like [Chef](https://www.chef.io/) that offer a similarly high quality experience. This article is more about the *pattern* as opposed to any individual implementation.

Benefits of this pattern:

* Supported - Puppet is a well established tool-chain with a large ecosystem of packages, documentation, and support.
* Cross platform - There are built-in primitives for making your configuration work on *any* operating system.
* Declarative - No more bash scripts! Use Puppet's declarative language to make simple statements about the state of your configuration/machine.
* *Idempotent* - This is the really special one. If you add a package to your setup, or a new file that needs to exist, you can *apply* your configuration, and only necessary changes will be made. No need to care about what it looked like when you ran it last, or what you've added since. Simply apply your configuration and Puppet handles the rest.

---

There are *tons* of articles out there about managing your infrastructure with puppet. However, taming your local setup is a different matter and is in general much simpler than most Puppet setups. I won't detail the full set up in this article (maybe in a future post by request), but we can get into a few examples of puppet code that has made my life a lot easier.

## General Applications

If you just want a taste of what this might look like, without looking at some dense configuration, this will serve as an example of installing a few applications. If you want to see some more complex configurations, then read on!

```puppet
class apps {
  package{'google-chrome':
    provider => homebrew
  }

  package{'spotify':
    provider => homebrew
  }

  package{'slack':
    provider => homebrew
  }

  package{'flycut':
    provider => homebrew
  }
}
```


## Shell/Zsh

There are a few differences between my Linux and OSX setups for my shell. Primarily, I use iterm2 with some custom fonts. Likewise, as you'll see throughout, installing packages on OSX requires a special homebrew `provider` key to be set. This is provided by a [third party dependency](https://forge.puppet.com/thekevjames/homebrew).

```puppet
class zsh{
  if $facts['os']['family'] == 'Darwin' {
    package { 'iterm2':
      provider => homebrew
    }

    package { 'zsh':
      provider => homebrew
    }

    package { 'wget':
      provider => homebrew
    }

    package{ 'caskroom/fonts':
      provider => tap
    }

    package{ 'font-firacode-nerd-font':
      require => Package['caskroom/fonts'],
      provider => homebrew
    }
  } else {
    package {'zsh':
    }

    package {'wget':
    }
  }


  exec {'install oh my zsh':
    command => '/usr/local/bin/wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh',
    require => [Package['zsh'], Package['wget']],
    user => $user,
    environment => ["HOME=${home}"]
  }

  vcsrepo { "${home}/.oh-my-zsh/custom/themes/powerlevel9k":
    ensure => latest,
    provider => git,
    source => 'https://github.com/bhilburn/powerlevel9k.git'
  }

  file { ["${home}/.zshrc.d", "${home}/.zshenv.d"]:
    ensure => 'directory',
    owner => $user
  }

  file { "${home}/.zshrc":
    ensure => link,
    target => "${puppet_path}/modules/zsh/files/.zshrc",
    owner => $user
  }

  file { "${home}/.zshenv":
    ensure => link,
    target => "${puppet_path}/modules/zsh/files/.zshenv",
    owner => $user
  }

  zsh::file { 'base.zsh':
    source_module_name => 'zsh',
    source_file_name => 'base.zsh',
    file_type => 'rc',
    zsh_file_name => 'base'
  }

  zsh::file { 'aliases.zsh':
    source_module_name => 'zsh',
    source_file_name => 'aliases.zsh',
    file_type => 'rc',
    zsh_file_name => 'aliases'
  }
}
```

* My `.zshrc` and `.zshenv` are both just simple files that source each file in their `.zshrc.d/` and `.zshenv.d/` directories. This lets any other module/manifest add code to my zshrc/zshenv by simply placing a file in that directory.
* By symlinking files directly back to the directory where they live in puppet, I can edit them via the symlink, so `vim ~/.zshrc` is the same as editing the file in the project.
* Anything that must be *explicitly* different between OSs is a simple if statement.

### zsh::file
This is an example of the kinds of abstractions you can get easily with puppet. Any one of my layers can use that class to place a file in `.zshrc.d/` or `.zshenv.d/`.
This lets me stop having to think about a few monolithic rc files and divide them up by purpose/domain instead. The class declration looks like this:

```puppet
define zsh::file ($source_module_name, $source_file_name, $zsh_file_name, $file_type) {
  $puppet_path = lookup(puppet::path)
  $home = lookup(users::home_dir)
  $user = lookup(users::username)

  if $file_type == "env" {
    $folder = ".zshenv.d"
  } elsif $file_type == "rc" {
    $folder = ".zshrc.d"
  } else {
    fail("${file_type} is not a valid zsh file type")
  }

  file { "${home}/${folder}/${zsh_file_name}.zsh" :
    ensure => link,
    target => "${puppet_path}/modules/${source_module_name}/files/${source_file_name}",
    owner => $user
  }
}
```

Now any layer that needs some rc magic can just add a flie and use `zsh::file{}` to get it to the right place!


## Emacs and Org File Synchronization
```puppet
class emacs {
  $home = lookup(users::home_dir)
  $puppet_path = lookup(puppet::path)
  $user = lookup(users::username)

  file { "${home}/Development":
    ensure => directory
  }

  vcsrepo { "${home}/.emacs.d/":
    ensure   => present,
    provider => git,
    source   => 'git://github.com/syl20bnr/spacemacs.git',
    revision => 'develop',
    owner => $user
  }

  file { "${home}/.spacemacs":
    ensure => link,
    target => "${puppet_path}/modules/emacs/files/.spacemacs",
    owner => $user
  }

  file { "${home}/spacemacs_plugins":
    ensure => link,
    target => "${puppet_path}/modules/emacs/files/spacemacs_plugins",
    owner => $user
  }

  if $facts['os']['family'] == 'Darwin' {
    package {'font-source-code-pro':
      provider => 'homebrew',
      require => Package['caskroom/fonts']
    }

    package { 'ag':
      ensure => present,
      provider => homebrew
    }

    package { 'd12frosted/emacs-plus':
      ensure   => present,
      provider => tap,
    }

    package { 'emacs-plus':
      require => Package['d12frosted/emacs-plus'],
      ensure   => present,
      provider => homebrew,
      install_options => ["--HEAD", "--with-natural-title-bars"]
    }

    file { '/Applications/Emacs.app':
      require => Package['emacs-plus'],
      ensure => link,
      target => "/usr/local/Cellar/emacs-plus/*/Emacs.app/",
      owner => $user
    }

    package { 'syncthing':
      provider => homebrew
    }

    exec { '/usr/local/bin/brew services start syncthing':
      environment => ["HOME=${home}"],
      require => Package['syncthing'],
      user => $user
    }

  } else {
    include apt

    package {'emacs':
    }

    apt::source { 'syncthing':
      location => 'https://apt.syncthing.net/',
      key => {
        id => '8735F5AF62A99A628EC13377B8F999C007BB6C57',
        source => 'https://syncthing.net/release-key.txt'
      }
    }

    package {'syncthing':
    }

    exec {'enable syncthing service':
      command => '/bin/systemctl enable syncthing@zacharydaniel.service',
      require => Package['syncthing']
    }

    exec {'/bin/systemctl start syncthing@zacharydaniel.service':
      require => Exec['enable syncthing service']
    }
  }
}
```

* Keeps me on the latest develop branch of spacemacs whenever I apply my setup
* Sets up syncthing, a tool I use to synchronize my org folder across my devices (so I can mark things as done on mobile, for example)
* Sets up an Emacs.app file that lets me launch emacs from spotlight
* Manages my plugins and configuration files

Hopefully the examples are helpful/useful. If you have trouble setting this kind of thing up yourself and need some help, let me know and I might write up a more specific/step-by-step guide. I could additionally hollow out my own setup and provide it as a template/skeleton.
