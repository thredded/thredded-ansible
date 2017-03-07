# Ansible Playbooks for running a Thredded App on Ubuntu 16.04

This is a starter set of [Ansible](https://www.ansible.com/) playbooks
for deploying a Thredded app to an Ubuntu 16.04 server.

**Warning: Experimental, there be dragons.**

These playbooks provide a good starting point if you're looking to deploy
Thredded to a VPS or to a bare-metal server.

TODO:

1. Email via postfix.
2. Backups via [backup](https://github.com/backup/backup).
3. Start the app's services on reboot (need to add a custom foreman template).

## Table of Contents

* [Overview](#overview)
   * [Provisioning](#provisioning)
   * [Deployment](#deployment)
   * [App compatibility](#app-compatibility)
* [Usage](#usage)
* [Deploying to VirtualBox](#deploying-to-virtualbox)
* [Deploying to production](#deploying-to-production)
   * [Adding the server to the hosts inventory](#adding-the-server-to-the-hosts-inventory)
   * [Creating a user for provisioning](#creating-a-user-for-provisioning)
   * [Provisioning](#provisioning-1)
   * [Deploying](#deploying)
   * [Configure HTTPS](#configure-https)
   * [Enable backups](#enable-backups)
   * [Configure postfix](#configure-postfix)
* [How to](#how-to)
   * [Deploy a new version of the app to the production server](#deploy-a-new-version-of-the-app-to-the-production-server)
   * [Update the app's environment variables](#update-the-apps-environment-variables)
   * [See the logs](#see-the-logs)
   * [Open a rails console on a production server](#open-a-rails-console-on-a-production-server)
* [Tips](#tips)
* [Credits](#credits)

## Overview

There are two playbooks included: `provision.yml` and `deploy.yml`.

### Provisioning

The provisioning playbook must be run as a user with password-less `sudo` access.

This playbook ensures the following:

Via the `webapp` role:
* System-wide dependencies are installed and are up-to-date.
* The app-specific user exists.
* The app directory (`/var/www/$APP` by default) exists and belongs to the
  app-specific user.

Via the `db` role:
* The Postgresql database server is set up and has a user for the app.

Via the `memcached` role:
* The memcached server is set up (by default, on the same machine(s) as
  the webserver(s)).

Via the `nginx` role:
* The nginx webserver is set up and the app's site configuration is up to date.

Via the `foreman_systemd` role:
* The app services are managed by `systemd` and their configuration
  is exported via [foreman](http://ddollar.github.io/foreman/) from `files/Procfile`.
* The app-specific user has permissions to start and stop app services.

Via the `letsencrypt` role (production only):
* Let's Encrypt is configured to provide and automatically renew the SSL
  certificate for the app's webserver.

### Deployment

The deployment playbook is run as the app-specific user.

This playbook ensures the following via the `deploy-rails` role:

* The configured Ruby version is installed (via `rbenv`).
* The app's environment variables are up to date (set in `~/.pam_envinoment`).

The playbook then fetches the Rails app code from the configured git repository,
compiles the assets, runs the migrations, and restarts the app services.

### App compatibility

These playbooks work out of the box with apps created with
thredded_create_app v0.1.8+.

If your app was created with an older version of thredded_create_app,
or was not created with thredded_create_app, you'll need to:

1. Copy the [config/puma.production.rb](https://raw.githubusercontent.com/thredded/thredded_create_app/master/lib/thredded_create_app/tasks/production_configs/puma.production.rb) file.
2. Add the following snippet to the `config/environments/production.rb` file:
   ```ruby
   if ENV['MEMCACHE_SERVERS']
     config.cache_store = :dalli_store,
         ENV['MEMCACHE_SERVERS'].split(','), {
             namespace: ENV['APP_ID'] || raise("ENV['APP_ID'] not set"),
             socket_timeout: 1.5,
             socket_failure_delay: 0.2,
             down_retry_delay: 60,
             pool_size: [2, ENV.fetch('WEB_CONCURRENCY', 3).to_i *
                            ENV.fetch('MAX_THREADS', 5).to_i].max
         }
   end
   ```
3. Ensure that the `config/database.yml` file uses `ENV['DATABASE_URL']` in production, i.e.:
  ```yaml
  production:
    url: <%= ENV['DATABASE_URL'].inspect if ENV['DATABASE_URL'] %>
  ```

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

## Deploying to VirtualBox

First, install VirtualBox v5.0+ and Vagrant v1.9+.

Then, create a new instance with:

```bash
vagrant up
```

Then, **provision** the VM instance:

```bash
ansible-playbook provision.yml -e "config=vars/${APP}.yml" -u ubuntu --private-key=.vagrant/machines/vm1/virtualbox/private_key
```

*If you get an SSH error, verify that you can
[SSH into the host as the `ubuntu` user](#tips)*.

Now, add the app user SSH key to the app's Git repository deploy keys.

1. Print the key with:

    ```bash
    ansible webservers -u $APP -a 'cat ~/.ssh/id_rsa.pub'
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
ansible-playbook deploy.yml -e "config=vars/${APP}.yml"
```

This will run the migrations, but you might also want to seed the database
if deploying for the first time. You can do so by running:

```bash
ansible 'webservers[0]' -u $APP -a "chdir=/var/www/$APP/current bundle exec rails db:seed"
```

Congratulations! You can now open the app at http://localhost:8080 and log in as
`admin@$APP.com` with password `123456`.

## Deploying to production

The guide describes provisioning deploying to a single production server.

The process is largely identical to deploying to Vagrant, but we will also:

1. Enable HTTPS via letsencrypt with automated certificate renewal.
2. Enable daily database backups using [backup]. **TODO**
3. Configure the [postfix] mail transfer agent to send email. **TODO**

This guide focuses on the differences from the VirtualBox guide above and
assumes that you have already followed the VirtualBox guide.

[backup]: https://github.com/backup/backup
[postfix]: http://www.postfix.org/

### Adding the server to the hosts inventory

First, we need to create the hosts inventory file for our production hosts:

1. Copy the example hosts file:

   ```bash
   cp hosts/example hosts/${APP}-prod
   ```

2. Set the host IP address in the `hosts/${APP}-prod` file.

We will need to tell Ansible to use this inventory file by passing
`-i hosts/${APP}-prod` to every `ansible` and `ansible-playbook` command.
Alternatively, you can change the default inventory file in `ansible.cfg`.

### Creating a user for provisioning

If you've created a new server with something like [Scaleway](https://scaleway.com)
(offers a 3 € a month VPS large enough for Thredded) then the server by default
will only have a root user with SSH key authentication, so you can skip this step.

**If** you do not have password-less SSH-as-root, or you'd prefer to not use
the root user (e.g. if you work in a team), you can create another user instead
like this:

1. Create a user with the same name as your local user on the production servers.
   Omit the `password` argument if you're sure you'll never lose your SSH key.

   ```bash
   ansible all -i hosts/${APP}-prod -u root -m user -a \
     "name=$USER groups=sudo password=$(mkpasswd --method=sha-512)"
   ```

2. Authorize your SSH key to connect as `$USER`:

   ```bash
   ansible all -i hosts/${APP}-prod -u root -m authorized_key -a \
     "user=$USER key={{lookup('file', '~/.ssh/id_rsa.pub')}}"
   ```

3. Allow the newly created user to run `sudo` **without password**:

   ```bash
   ansible all -i hosts/${APP}-prod -u root -m lineinfile -a \
     "path=/etc/sudoers.d/$USER create=yes line='$USER ALL=(ALL) NOPASSWD:ALL' validate='/usr/sbin/visudo -cf %s'"
   ```

4. Pass `-u $USER` instead of `-u root` to Ansible in the
   *Provisioning* commands below.

### Provisioning

Copy the vars config file you've created earlier for Vagrant, as we'll need the
configuration to be slightly different (we'll change it afterwards).

```bash
cp vars/${APP}.yml vars/${APP}-prod.yml
```

You may want to change all the passwords and the secret key in the new config.

Then, run the provisioning playbook:

```bash
ansible-playbook provision.yml -u root -e "config=vars/${APP}-prod.yml" -i hosts/${APP}-prod
```

### Deploying

1. Add the SSH key of the app user to your git repository deploy keys.
   To print the key, run:

   ```bash
   ansible webservers -i hosts/${APP}-prod -u $APP -a 'cat ~/.ssh/id_rsa.pub'
   ```

2. Run the deployment playbook:

   ```bash
   ansible-playbook deploy.yml -i hosts/${APP}-prod -e "config=vars/${APP}-prod.yml"
   ```

3. Seed the database:

   ```bash
   ansible 'webservers[0]' -i hosts/${APP}-prod -u $APP -a "chdir=/var/www/$APP/current bundle exec rails db:seed"
   ```

Congratulations! You can now open the app at the server's IP and log in as
`admin@$APP.com` with password `123456`. Change the password immediately.

### Configure HTTPS

Next, we will configure HTTPS with Let's Encrypt, including certificate
auto-renewal.

For this, you will need to have `{{app_host}}` and `www.{{app_host}}`
configured as the A-records for your server IP.

First, in the `vars/${APP}-prod.yml` file, set the `app_host` and uncomment the
*HTTPS with Let's encrypt* section.

Then, run the provisioning playbook:

```bash
ansible-playbook provision.yml -u root -e "config=vars/${APP}-prod.yml" -i hosts/${APP}-prod
```

That's it! Verify that `https://{{app_host}}` works, and that these variations
all redirect to that URL: `http://{{app_host}}`, `http://www.{{app_host}}`,
and `https://www.{{app_host}}`.

Test the quality of your HTTPS configuration at [SSL Labs](https://ssllabs.com/ssltest/).

### Enable backups

TODO

### Configure postfix

TODO

## How to

### Deploy a new version of the app to the production server

Run the deployment playbook:

```bash
ansible-playbook deploy.yml -i hosts/${APP}-prod -e "config=vars/${APP}-prod.yml"
```

### Update the app's environment variables

Update the `vars/$APP.yml` file and either do a full deploy, or run:

```bash
ansible-playbook deploy.yml -e "config=vars/${APP}.yml" --tags env
```

The command above will only update the environment and then restart the app.

**NB**: Removing an environment variable from `vars/$APP.yml` does **not**
remove it from the environment file on the server (`~$APP/.pam_environment`).

### See the logs

The application and webserver logs are located at `/var/www/$APP/shared/log`.
The app service output (such as errors when starting the services)
is logged to `/var/log/syslog`.

### Open a `rails console` on a production server

In production:

```bash
# Replace "${IP}" with the server's IP address
ssh ${APP}@${IP} -t "cd /var/www/$APP/current && bundle exec rails c"
```

To print a server IP address / hostname:

```bash
ansible 'webservers[0]' -i hosts/${APP}-prod -m debug -a 'var=ansible_host' | grep -oE "[0-9.]{2,}"
```

On VirtualBox:

```bash
ssh $APP@127.0.0.1 -p 2222 -t "cd /var/www/$APP/current && bundle exec rails c"
```

## Tips

You can ssh into the VirtualBox instance on port 2222:

```bash
# As the "ubuntu" user (this user can run sudo without a password)
ssh ubuntu@127.0.0.1 -p2222 -i .vagrant/machines/vm1/virtualbox/private_key
# As the app user
ssh $APP@127.0.0.1 -p2222
```

To run a command as the web user via Ansible:

```bash
# Use webservers[0] for just one of the servers
ansible webservers -u $APP -a "chdir=/var/www/$APP/current bundle exec rails db:seed"
```

To see the output of commands as they run,
pass the `-vvv` flag to `ansible` or `ansible-playbook`.

## Credits

* The Rails deployment role is based on: https://github.com/nicolai86/ansible-rails-deployment.
* Rbenv installation is based on: https://github.com/erasme/ansible-rbenv.
* The Nginx role is based on: https://github.com/jdauphant/ansible-role-nginx.
* The Memcached role is based on: https://github.com/geerlingguy/ansible-role-memcached.
* The Nginx HTTPS config is based on: http://stackoverflow.com/a/41948807/181228
* Let's encrypt HTTPS certs and auto-renewal are based on:
  * https://github.com/thefinn93/ansible-letsencrypt
  * https://gist.github.com/mattiaslundberg/ba214a35060d3c8603e9b1ec8627d349
