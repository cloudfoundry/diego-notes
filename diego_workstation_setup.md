# Diego Workstation Setup

# Required Dependencies

## Mac Apps
- Vagrant https://www.vagrantup.com/downloads.html
- VirtualBox https://www.virtualbox.org/wiki/Downloads
- CF cli https://github.com/cloudfoundry/cli/releases

## Homebrew

Install Howebrew:
```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Install the following packages.  Don't forget to look for instructions to complete manual steps as part of each install (like adding the sql db's to launchctl):
```bash
brew install git
brew install go --with-cc-common
brew install direnv
brew install mysql
brew install postgres
brew install ruby-install
brew install chruby
```

## Ruby Packages
```bash
ruby-install ruby 2.3.1
chruby ruby-2.3.1
```
Add the following to .bashrc:
```bash
source /usr/local/opt/chruby/share/chruby/chruby.sh
chruby ruby-2.3.1
```
Install the bosh cli and bundler:
```bash
gem install bosh_cli
gem install bundler
```

## Go Packages

`mkdir $HOME/go` add the following to .bash_profile:

```bash
export PATH=$PATH:$HOME/go/bin
export GOPATH=$HOME/go
```
Go get the following packages:
```bash
go get golang.org/x/tools/cmd/goimports
go get github.com/tools/godep
go get github.com/onsi/ginkgo/ginkgo
```

Install spiff from an official binary release found [here](https://github.com/cloudfoundry-incubator/spiff/releases)

## (Pivotal Workstations) Install git-secrets

Follow the instructions [here](https://github.com/pivotal-cf/sec-issues/blob/master/git-secrets.md).

# Diego Team Preferences

## Mac Apps
Install the following:
- iTerm2 https://www.iterm2.com/nightly/latest
- ShiftIt https://raw.github.com/onsi/ShiftIt/master/ShiftIt.zip
- Sublime Text 3 http://www.sublimetext.com/3
- Slack https://itunes.apple.com/app/slack/id803453959?ls=1&mt=12
- Wraparound http://www.macupdate.com/app/mac/19599/wraparound
- Flycut - Flycut https://github.com/TermiT/Flycut
- Docker Toolbox https://www.docker.com/docker-toolbox

## Bash-It
Install Bash-It with plugins and other garbage disabled.
```bash
git clone --depth=1 https://github.com/Bash-it/bash-it.git ~/.bash_it
~/.bash_it/install.sh --none
```

## Homebrew
Install Howebrew:
```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Install the following packages.  Don't forget to look for instructions to complete manual steps as part of each install (like adding the sql db's to launchctl):
```bash
brew install ack
brew install git-duet
brew install ag
brew install bash-completion
brew install python
brew install tmux
brew install tmate
brew install jq
brew install fasd
brew install aria2
```

## Go Packages

`mkdir $HOME/go` add the following to .bash_profile:

```bash
export PATH=$PATH:$HOME/go/bin
export GOPATH=$HOME/go
```

```bash
go get github.com/vito/boosh
```

## Python Packages

- pip install aws

## Sublime Packages

Install the package manager through the Sublime Text console. The console is accessed via the ``ctrl+` `` shortcut or the `View > Show Console` menu. Once open, paste the Python code found [here](https://packagecontrol.io/installation) into the console.

- Package Install: gosublime
- Package Install: spacegray
```json
// GoSublime Settings
{
	"fmt_cmd": ["goimports"]
}
```
```json
// User Settings
{
  "theme": "Spacegray Eighties.sublime-theme",
  "save_on_focus_lost": false
}
```


## Vim
Full instructions can be found at http://luansantos.com/vimfiles/ 

```bash
git clone --depth=1 http://github.com/luan/vimfiles.git ~/.vim
cd ~/.vim
./install
```

## Enable fasd
Add the following line to ~/.bash_profile:
```bash
eval "$(fasd --init auto)"
```

## Set up direnv hook
Add the following line to the end of ~/.bashrc:
```bash
eval "$(direnv hook bash)"
```

## Set up git
copy workstation ~/.gitconfig and ~/.git-authors and add the following to ~/.bash_profile:
```bash
export GIT_DUET_GLOBAL=true
export GIT_DUET_ROTATE_AUTHOR=true
```

## Copy system aliases
Add the following to ~/.bash_profile:
```bash
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
```

Copy workstation aliases from `~/emacs-config/cookbooks/development/files/default/alias.sh` to ~/.bash_aliases and define additional aliases in ~/.bash_aliases

## Set up arrow-up partial bash completion
```bash
echo '# arrow up
"\e[A":history-search-backward
## arrow down
"\e[B":history-search-forward' > ~/.inputrc
```

## Create a docker-machine
```
docker-machine create default # --engine-insecure-registry=(($diego-docker-cache-ip))
```
