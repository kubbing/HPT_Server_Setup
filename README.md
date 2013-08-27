# First-time-login Steps

Login as `root`.

Update system.

	# aptitude update && aptitude upgrade
	
Install basic editor, git, browser and favorite shell.
	
	# aptitude install vim htop git mc lynx zsh
	
Change Hostname
	
- Change hostname in `/etc/hostname` to your needs.
- Change hostname to the new one in `/ets/hosts` (line with `127.0.0.1`).
- Changes apply after reboot.
	
## Automatic Security Updates

	# aptitude install unattended-upgrades
	# dpkg-reconfigure -plow unattended-upgrades [y]
	
## Add User

Add user and set password.

1. `# useradd -m -d /home/<user> -s /bin/zsh <user>`
- `# passwd <user>`

Add user to sudo. 

- `# visudo` 
- add line `<user> ALL=(ALL:ALL) ALL`
- save as `/etc/sudoers`

Login as new user.
 
- `# su <user>`

Select to populate `.zshrc` with reasonable defaults.

## Set Up Zsh

This is (imo) reasonable detaults in `~/.zshrc`.

	export LC_CTYPE=en_US.UTF-8
	export LC_ALL=en_US.UTF-8

	autoload -U colors && colors

	autoload -Uz promptinit
	PROMPT="%n%{${fg[cyan]}%}@%{$reset_color%}%m %{${fg[yellow]}%}%1~%{$reset_color%} %{${fg[white]}%}$%{$reset_color%} "
	promptinit

	setopt histignorealldups sharehistory

	# Use emacs keybindings even if our EDITOR is set to vi
	bindkey -e

	# Keep 1000 lines of history within the shell and save it to ~/.zsh_history:
	HISTSIZE=1000
	SAVEHIST=1000
	HISTFILE=~/.zsh_history

	# Use modern completion system
	autoload -Uz compinit
	compinit

	zstyle ':completion:*' auto-description 'specify: %d'
	zstyle ':completion:*' completer _expand _complete _correct _approximate
	zstyle ':completion:*' format 'Completing %d'
	zstyle ':completion:*' group-name ''
	zstyle ':completion:*' menu select=2
	eval "$(dircolors -b)"
	zstyle ':completion:*:default' list-colors ${(s.:.)LS_COLORS}
	zstyle ':completion:*' list-colors ''
	zstyle ':completion:*' list-prompt %SAt %p: Hit TAB for more, or the character to insert%s
	zstyle ':completion:*' matcher-list '' 'm:{a-z}={A-Z}' 'm:{a-zA-Z}={A-Za-z}' 'r:|[._-]=* r:|=* l:|=*'
	zstyle ':completion:*' menu select=long
	zstyle ':completion:*' select-prompt %SScrolling active: current selection at %p%s
	zstyle ':completion:*' use-compctl false
	zstyle ':completion:*' verbose true

	zstyle ':completion:*:*:kill:*:processes' list-colors '=(#b) #([0-9]#)*=0=01;31'
	zstyle ':completion:*:kill:*' command 'ps -u $USER -o pid,%cpu,tty,cputime,cmd'

## Set Up Vim

Minimalistic setup in `~/.vimrc`.
	
	" basics
	
	set nocompatible
	syntax on
	
	filetype plugin indent on
	set autochdir
	set backspace=indent,eol,start
	
	set mouse=a
	
	set wildmenu
	
	" vim ui
	set incsearch
	set number
	set numberwidth=4
	set tabstop=4
	set ruler

## Public Key Login
	
- Generate keys on local machine in using `$ ssh-keygen -t rsa` (preferably in `~/.ssh`).
- Check that your new user on server has `~/.ssh` folder create. If not, create it.
- Still on local machine: `scp` your `id_rsa.pub` to `user@server:~/.ssh/authorized_keys`
- Now edit file `/etc/ssh/sshd_config` to disable password authentication:

Change lines:

	PermitRootLogin no
	PubkeyAuthentication yes
	AuthorizedKeysFile     %h/.ssh/authorized_keys
	PasswordAuthentication no
	TCPKeepAlive yes
	TCPKeepAlive yes
	ClientAliveInterval 60
	ClientAliveCountMax 10
	UsePAM no
	
- `service ssh reload`
- Change your local `~/.ssh/config`:

Add this entry:

	Host server
	HostName	<ip_address>
	User		<user>
	IdentityFile ~/.ssh/id_rsa.pub
	
- Now login as new user `ssh server`
- Profit!
	
# Ruby, Nginx, Passenger

## Install Prerequisites

This took less than 20s on my server :).

	# aptitude install build-essential libssl-dev zlib1g-dev libcurl4-openssl-dev imagemagick libmagickwand-dev

## Create `deploy` User

	# useradd -m -d /home/deploy -s /bin/zsh deploy
	
- You can setup `.zshrc` and `.vimrc` if you want.
- You have to add `deploy` user to sudoers (`# visudo`) to install passenger.	
- Set password for `deploy` user.
- Login as `deploy`.

## Install rbenv, ruby-build

As `deploy`:

**Note:** put rbenv scripts into `.zshenv`; this way they will be available in all shells.

- Install rbenv using guide [github](https://github.com/sstephenson/rbenv/).
- Install rbenv-build using guide [github](https://github.com/sstephenson/ruby-build).
	
## Install Ruby

As `deploy` in `~`:

	$ rbenv install -l
	$ rbenv install <version>

## Install Passenger and Nginx

As `deploy` in `~`:

- `gem list passenger --remote`
- `$ gem install passenger`
- `$ sudo which passenger-install-nginx-module`
- Install Ubuntu service for passenger using [github](https://github.com/chloerei/nginx-init-ubuntu-passenger).

### Test Nginx

As `deploy` in `~`:

- Add `deploy` user to `www-data` grou by editing `/etc/group`.
- `# mkdir -p /var/www/default`
- `# chown -R www-data:www-data /var/www`
- `# chmod -R g+w /var/www`
- `$ echo 'it works' > /var/www/default/index.html`

Add default Nginx virtualhost (`/opt/nginx/config/nginx.conf`):

	server {
	      listen 80 default;
	      server_name <server_name>;
	      root /var/www/default;
	}
	
And edit `/etc/hosts` on local computer:

	<server_ip> <server_name>
	
And test from local computer `$ curl <server-name>`. You should get `it works` response.
	
## Ruby on Rails (finally!)

As `deploy` in `~`:

- `$ gem install rails`
- `$ rbenv rehash` 

As `deploy` in `/var/www` create and test new rails application:

- Test using `rails new test.rails`.
- `cd` into new app and uncomment `therubyracer` line in `Gemfile`.
- `bundle install`
- `$ rails generate controller hello index`
- Edit `config/routes.rb`, add line `root 'hello#index'`.

Add nginx virtualhost, and reload nginx (`sudo service nginx reload`).

	server {
	      listen 80;
	      server_name test.rails;
	      root /var/www/test.rails/public;
	      passenger_enabled on;
	}
	
And add local computer hosts entry (`/etc/hosts`).

	<server_ip> test.rails
	
Test using `$ curl 'test.rails'` from local machine. You should receive couple of HTML lines of your new app.
	
# Database PostgreSQL

## Install

Install database and create `deploy` user.

	# apt-get install postgresql libpq-dev
	# su postgres
	$ createuser deploy [n/y/n]

- Edit `/etc/postgresql/9.1/main/postgresql.conf`, uncomment `listen_addresses = 'localhost'` line.
- Edit `/etc/postgresql/9.1/main/pg_hba.conf`

Change lines (change localhosts to `trust`):

	# "local" is for Unix domain socket connections only
	local   all             all                                     trust
	# IPv4 local connections:
	host    all             all             127.0.0.1/32            trust
	# IPv6 local connections:
	host    all             all             ::1/128                 trust

- `service postgresql restart`
	
## Test	

 As `deploy`:

- `cd` into your `test.rails` app in `/var/www/test.rails`.
- Add `gem 'pg'` to your `Gemfile`
- `bundle install`
- `rails generate scaffold Article title:string description:text`
- Edit database config.

New content of `config/database.yml` (**Note:** indent using spaces!):

	development:
  	  adapter: postgresql
  	  encoding: unicode
  	  database: test_development
  	  host: localhost
  	  pool: 5
  	  timeout: 5000

	# Warning: The database defined as "test" will be erased and
	# re-generated from your development database when you run "rake".
	# Do not set this db to the same as development or production.
	test:
  	  adapter: postgresql
  	  encoding: unicode
  	  database: test_testing
  	  host: localhost
  	  pool: 5
  	  timeout: 5000

	production:
  	  adapter: postgresql
  	  encoding: unicode
  	  database: test_production
  	  host: localhost
  	  pool: 5
  	  timeout: 5000
  	  
Create database and run migrations:
  	  
- `RAILS_ENV=production rake db:create`
- `RAILS_ENV=production rake db:migrate`
- `sudo service nginx restart`
- Test using `$ curl 'test.rails/articles.json'` from local machine. You should receive empty JSON array `[]`.

# Capistrano Deployment

As `deploy` on server:

- `cd` into `~/.ssh`
- `ssh-keygen -t rsa`
- Upload `id_rsa.pub` to you git server ([github](http://www.github.com), [bitbucket](http://www.bitbucket.com))

As `deploy` on local machine:

- `cd` into your `test.rails` app in `/var/www/test.rails`.
- Uncomment `capistrano` line in `Gemfile`
- `bundle install`
- `capify i`
- Uncomment `assets` line in `Capfile`

Edit `config/deploy.rb` with following:

	require "bundler/capistrano"

	set :scm, :git
	set :repository, "git repo url similar to git@bitbucket..."
	set :branch, "master"

	set :domain, "<server_ip>"
	set :rails_env, "production"
	set :application, "<you_app_folder_name>"
	set :deploy_to, "/var/www/#{application}"
	
	# This avoids deletion of `uploads` folder after deployment
	set :shared_children, shared_children + %w{public/uploads} 

	set :user, :deploy # our `deploy` user
	set :use_sudo, false

	server "#{domain}", :app, :web, :db, primary: true

	# If you are using Passenger mod_rails uncomment this:
	namespace :deploy do
  		task :start do ; end
  		task :stop do ; end
  		task :restart, :roles => :app, :except => { :no_release => true } do
    		run "#{try_sudo} touch #{File.join(current_path,'tmp','restart.txt')}"
  		end
	end

	namespace :deploy do
		desc '*DANGER* Same as cold, but creates, migrates and seeds database correctly'
		task :cold_db do
			update
			db_create
			db_migrate
			db_seed
			start
		end
	end

	namespace :deploy do
		desc '*DANGER* Drops, recreates, migrates and seeds database'
		task :db_all do
		end

		desc '*DANGER* Drops database'
		task :db_drop do
		 	run "cd #{current_path}; RAILS_ENV=#{rails_env} bundle exec rake db:drop;"
		end

		desc '*DANGER* Creates database'
		task :db_create do
		 	run "cd #{current_path}; RAILS_ENV=#{rails_env} bundle exec rake db:create;"
		end

		desc '*DANGER* Migrates database'
		task :db_migrate do
		 	run "cd #{current_path}; RAILS_ENV=#{rails_env} bundle exec rake db:migrate;"
		end

		desc '*DANGER* Seeds database'
		task :db_seed do
		 	run "cd #{current_path}; RAILS_ENV=#{rails_env} bundle exec rake db:seed;"
		end
	end

	after "deploy:db_all", "deploy:stop", "deploy:db_drop", "deploy:db_create", "deploy:db_migrate", "deploy:db_seed", "deploy:start"
	
Now you are ready to deploy your application.

- `cap deploy:setup`
- `cap deploy:check`
- `cap deploy:cold_db`

The End.




	
	
	

	