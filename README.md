# All-in-one Ansible Playbooks for running a Thredded App on Ubuntu 16.04

This is a starter set of playbooks for provision and deploying a Thredded app.

**Warning: Experimental work-in-progress, there be dragons.**

## Overview

There are two playbooks included: `provision.yml` and `deploy.yml`.

### Provisioning

The provisioning playbook is run as a user with password-less `sudo` access.
It ensures the following:

* System-wide dependencies are installed and up-to-date.
* The Postgresql database server is set up and has a user for the app.
* The nginx webserver is set up and the app's site configuration is up to date.
* The app-specific user exists and has permissions to manage app services.
* The app services are managed by `systemd` and their configuration
  is exported via [foreman](http://ddollar.github.io/foreman/) from `files/Procfile`.
* The app user is granted permissions to start and stop the app services.

### Deployment

The deployment playbook is run as the app-specific user.
It ensures the following:

* The configured Ruby version is installed (via `rbenv`).
* The app's environment variables are up to date (set in `~/.pam_envinoment`).

It the fetches the Rails app code from the configured git repository,
compiles the assets, runs the migrations, and restart the app services.

### App compatibility

These playbooks work out of the box with apps created with
thredded_create_app v0.1.8+.

If your app was created with an older version of thredded_create_app,
or was not created with thredded_create_app, you'll need to copy the latest
[config/puma.production.rb](https://raw.githubusercontent.com/thredded/thredded_create_app/master/lib/thredded_create_app/tasks/production_configs/puma.production.rb)
file for this to work.

## Usage

First, clone this repo.

```bash
git clone https://github.com/thredded/thredded-ansible myapp-ansible
cd myapp-ansible
```

Then, copy the example variables file:

```bash
# Set the APP env var to your app name. Will be used throughout the Readme.
APP=myapp
# Copy the example variables file:
cp vars/example.yml "vars/${APP}.yml"
```

Then, set the required variables in the variables file.
Also, have a look at the `vars/defaults.yml` for the optional variables and
their defaults.

Finally, see the section below to try out your setup in a VM.

Once you've confirmed it's working in a VM, you can proceed to set up and deploy
to a real server or a VPS. **The Readme section for this is not ready yet.**

## VirtualBox installation

First, install VirtualBox v5.0+ and Vagrant v1.9+.

First, create a new instance with:

```bash
vagrant up
```

Then, **provision** the VM instance:

```bash
ansible-playbook provision.yml -e "app_name=$APP" -u ubuntu --private-key=.vagrant/machines/vm1/virtualbox/private_key
```

*If you get an SSH error, verify that you can [SSH into the host](#tips)*.

Now, add the app user SSH key to the app's Git repository deploy keys.

1. Print the key with:

    ```bash
    ansible all -u $APP -a 'cat ~/.ssh/id_rsa.pub'
    ```

2.  Add the key to the deploy keys in your repository settings.

    On GitLab, the deploy keys settings page is at:

    ```
    https://gitlab.com/$USER/$APP/deploy_keys
    ```

    On GitHub, it's at:

    ```
    https://github.com/$USER/$APP/settings/keys
    ```

Then, **deploy** to the VM instance:

```bash
ansible-playbook deploy.yml -e "app_name=$APP"
```

This will run the migrations, but you might also want to seed the database
if deploying for the first time. You can do so by running:

```bash
ansible 'webservers[0]' -u $APP -a "chdir=/var/www/$APP/current bundle exec rails db:seed"
```

Congratulations! You can now open the app at http://localhost:8080 and log in as
`admin@$APP.com` with password `123456`.

## How to

### See the logs

The logs are located at `/var/www/$APP/shared/log`.
The app service output is logged to `/var/log/syslog`.

### Update app environment variable

Update the `vars/$APP.yml` file and either do a full deploy, or run:

```bash
ansible-playbook deploy.yml -e "app_name=$APP" --tags env
```

The command above will only update the environment and then restart the app.

**NB**: Removing an environment variable from `vars/$APP.yml` does **not**
remove it from the environment file on the server (`~$APP/.pam_environment`).

### Open a `rails console` on a production server

```
ssh $APP@127.0.0.1 -l -c 'cd /var/www/
```

#### Tips

You can ssh into the VM instance on port 2222:

```bash
# As the app user
ssh $APP@127.0.0.1 -p2222
# As the "ubuntu" user (that can run sudo without a password
ssh ubuntu@127.0.0.1 -p2222 -i .vagrant/machines/vm1/virtualbox/private_key
```

You can run a command as the web user via ansible like this:

```bash
# Use webservers[0] for just one of the servers
ansible webservers -u $APP -a "chdir=/var/www/$APP/current bundle exec rails db:seed"
```

To see the output of commands as they run,
pass the `-vvv` flag to `ansible` or `ansible-playbook`.

## Credits

The Rails deployment role is based on: https://github.com/nicolai86/ansible-rails-deployment
Rbenv installation is based on: https://github.com/erasme/ansible-rbenv
The Nginx role is based on: https://github.com/jdauphant/ansible-role-nginx
