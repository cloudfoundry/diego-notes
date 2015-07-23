# Diego Workstation Setup

## Mac Apps
Install the following:
- iTerm2 https://www.iterm2.com/nightly/latest
- ShiftIt https://raw.github.com/onsi/ShiftIt/master/ShiftIt.zip
- Sublime Text 3 http://www.sublimetext.com/3
- Vagrant https://www.vagrantup.com/downloads.html
- VirtualBox https://www.virtualbox.org/wiki/Downloads
- Slack https://itunes.apple.com/app/slack/id803453959?ls=1&mt=12
- Spiff (install binary at `/usr/local/bin/spiff`) https://github.com/cloudfoundry-incubator/spiff/releases
- Wraparound http://www.macupdate.com/app/mac/19599/wraparound

## Bash-It
Install Bash-It with plugins and other garbage disabled.
```
git clone --depth=1 https://github.com/Bash-it/bash-it.git ~/.bash_it
~/.bash_it/install.sh --none
```

## Homebrew
Install Howebrew:
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Install the following packages.  Don't forget to look for instructions to complete manual steps as part of each install (like adding the sql db's to launchctl):
```
brew install git
brew install ruby-install
brew install chruby
brew install go --with-cc-common
brew install mysql
brew install postgres
brew install direnv
brew install ack
brew install ag
brew install bash-completion
brew install python
brew install tmux
brew tap nviennot/tmate
brew install tmate
brew install jq
brew tap git-duet/tap && brew install git-duet
brew install fasd
brew install aria2
```

### Brew Cask
- brew install caskroom/cask/brew-cask
- brew cask install flycut

## Ruby Packages
```
ruby-install ruby 2.1.6
chruby ruby-2.1.6
```
```
gem install bosh_cli
gem install bundler
```

## Go Packages

`mkdir $HOME/go` add the following the following to .bash_profile:

```
export PATH=$PATH:$HOME/go/bin
export GOPATH=$HOME/go
```

```
go get golang.org/x/tools/cmd/goimports
go get github.com/vito/boosh
go get github.com/vito/spiff
go get github.com/tools/godep
```

## Python Packages

- pip install aws

## Sublime Packages

Install the package manager through the Sublime Text console. The console is accessed via the ``ctrl+` `` shortcut or the `View > Show Console` menu. Once open, paste the following Python code into the console.

```
import urllib.request,os,hashlib; h = 'eb2297e1a458f27d836c04bb0cbaf282' + 'd0e7a3098092775ccb37ca9d6b2e4b7d'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

- Package Install: gosublime
- Package Install: spacegray
```
GoSublime Settings
{
	"fmt_cmd": ["goimports"]
}
```
```
User Settings
{
  "theme": "Spacegray Eighties.sublime-theme",
  "save_on_focus_lost": false
}
```


## Vim
Full instructions can be found at http://luansantos.com/vimfiles/ 

```
git clone --depth=1 http://github.com/luan/vimfiles.git ~/.vim
cd ~/.vim
./install
```

## Enable fasd
Add the following line to ~/.bash_profile:
```
eval "$(fasd --init auto)"
```

## Set up git
copy workstation ~/.gitconfig and ~/.git-authors

## Set up arrow-up partial bash completion
```
echo '# arrow up
"\e[A":history-search-backward
## arrow down
"\e[B":history-search-forward' > ~/.inputrc
```
