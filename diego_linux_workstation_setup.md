# Linux Workstation Setup

## Required Dependencies

1. Install common dependencies.
```
sudo apt-get install direnv git vagrant virtualbox
```

2. Install golang.

```
wget https://storage.googleapis.com/golang/go1.6.2.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.6.2.linux-amd64.tar.gz
```

Also add the following to your `.profile`:

```
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH=$PATH:$HOME/go/bin
```

3. Install golang packages.

```bash
go get golang.org/x/tools/cmd/goimports
go get github.com/vito/spiff
go get github.com/tools/godep
```

4. Install Ruby and Chruby.

```
wget -O ruby-install-0.6.0.tar.gz https://github.com/postmodern/ruby-install/archive/v0.6.0.tar.gz
tar -xzvf ruby-install-0.6.0.tar.gz
cd ruby-install-0.6.0/
sudo make install

wget -O chruby-0.3.9.tar.gz https://github.com/postmodern/chruby/archive/v0.3.9.tar.gz
tar -xzvf chruby-0.3.9.tar.gz
cd chruby-0.3.9/
sudo make install

ruby-install ruby
```

Add the following to your `~/.bashrc`:

```
source /usr/local/share/chruby/chruby.sh
chruby LATEST_RUBY_VERSION
```

5. Install ruby gems.

```bash
gem install bosh_cli
gem install bundler
```


## Diego Team Preferences

1. Install Nvidia Graphics Drivers (optional)
```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update
sudo apt-get install nvidida-364
```

2. Install common dependencies.
```
sudo apt-get install vim-nox tmux
```

3. Install vim config

```
git clone https://github.com/luan/vimfiles ~/.vim && ~/.vim/install
```

4. Install tmux config

```
curl -o ~/.tmux.conf https://raw.githubusercontent.com/luan/dotfiles/master/tmux.conf
```

In order to have the ssh-agent configured correctly when using tmux, add the following to your `.profile`:

```
if [ ! -S ~/.ssh/ssh_auth_sock ]; then
    eval `ssh-agent`
    ln -sf "$SSH_AUTH_SOCK" ~/.ssh/ssh_auth_sock
fi
export SSH_AUTH_SOCK=~/.ssh/ssh_auth_sock
```

When launching tmux, you should use `tmux -2` as it will respect the terminal configuration.

5. Install terminator

Terminator is a terminal emulator that is more similar to iTerm2 than the gnome terminal.

```
sudo apt-get install terminator
```
6. Install Chrome and Chrome Remote Desktop

```
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb

wget https://dl.google.com/linux/direct/chrome-remote-desktop_current_amd64.deb
sudo dpkg -i chrome-remote-desktop_current_amd64.deb
```

7. Install git-duet
Download from [releases page](https://github.com/git-duet/git-duet/releases).

```
sudo tar -C /usr/local/bin -xzf linux_amd64.tar.gz
```

8. **Pivotal Workstations** Install git-secrets

Follow the instructions [here](https://github.com/pivotal-cf/sec-issues/blob/master/git-secrets.md).

9. Install Steam

```
wget https://steamcdn-a.akamaihd.net/client/installer/steam.deb
sudo dpkg -i steam.deb
```
