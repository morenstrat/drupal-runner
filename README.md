# Drupal Runner - GitLab Review Apps with Drupal

Drupal Runner is a set of bash scripts that implement GitLab Review Apps for Drupal. It creates two static environments - from master and develop branch - and dynamic environments for all other branches. The source environment from which the dynamic branches are created can be configured int the `.gitlab-ci.yml` file.

## Requirements

* GitLab Server
* GitLab Runner
* Linux with sudo, lvm2 and rsync
* Wildcard DNS (*.example.org)
* Apache or Nginx Webserver
* MySQL or MariaDB Database server
* composer, drush

## Installation

Copy the scripts in the `bin` folder of this repository to a location that is within your `gitlab-runner` user's path.
Copy the configuration file in the `etc` folder of this repository to the `/etc` folder of the server where GitLab Runner is installed.

## Server Configuration

### Drupal Runner

Edit the configuration file to change the location of the Review Apps or the database and logical volume prefixes.

### GitLab Runner

The GitLab Runner must be registered with the `Shell` executor. Other executors are planned to be added in the future.

### Apache

Add a `ServerAlias` and `VirtualDocumentRoot` directive to your virtual host, e.g.:

```
<VirtualHost *:80>
  ServerAlias *.gitlab-runner.test
  VirtualDocumentRoot /var/www/%1/web
</VirtualHost>
```

For the server alias you need a matching wildcard DNS record. The virtual document root has to match the settings in your Drupal Runner configuration.

### MySQL / MariaDB

The gitlab-runner user must be able to create and delete databases without interaction. Execute the following snippet on your database server:

```
CREATE USER 'gitlab-runner'@'localhost';
GRANT ALL PRIVILEGES ON  `dr\_%` . * TO 'gitlab-runner'@'localhost';
```

Again, make sure that the prefix in this snippet (dr_) matches the Drupal Runner settings.

### Sudo

The gitlab-runner user must be able to perform certain actions with super user privileges. The drupal-runner-sudo bash script handles all these actions. Create a sudo config file so that it can be executed as super user by the gitlab-runner user, for instance:

```
/etc/sudoers.d/gitlab-runner:

gitlab-runner ALL=(ALL) NOPASSWD: /usr/local/bin/drupal-runner-sudo
```

Make sure to set the correct path to the script (/usr/local/bin in this case).

### Logical Volume Manager

Drupal Runner creates snapshots of a configurable source environment Drupal files directory in order to minimize the required hard disk space. The initial size of the snapshots is relatively small to save further space. In order to automatically extend the snapshot sizes, change the following lvm configuration:

```
/etc/lvm/lvm.conf:

activation {
  snapshot_autoextend_threshold = 70
  snapshot_autoextend_percent = 20
```

This makes sure that once 70 percent of a snapshot are used, it is increased by 20 percent.

## Client / CI-Configuration

In your repository, you will need a `.gitlab-ci.yml` file. There is a template at https://github.com/morenstrat/drupal-project - but be warned, this is work in progress.

A CLI for the client side of Drupal Runner is in the making. In the meantime, the necessary steps to set up your repository for Drupal Runner, should be executed manually. These steps highly depend on your local environment, but you should roughly perform the following steps:

* Create the proejct: `composer create-project morenstrat/drupal-project:8.x-dev some-dir --stability dev --no-interaction`
* Set the review domain in your `.gitlab-ci.yml` file (the wildcard DNS record)
* Create a git repository and push it to your GitLab server (`git init, git add ., git commit -m "Initial commit.", git remote add origin REPO-URL, git push -u origin master`)
* Drupal Runner will now install Drupal
* Once Drupal is installed, got the created environment and copy the environment ID
* Now add the environment ID to the `.gitlab-ci.yml` file
* Also edit the `drush/sites/self.site.yml` file and add the missing configuration
* Setup yout local database connection in `web/sites/default/settings.local.php`
* Import the database from the master Review App: `drush sql:sync @self.master @self`
* IMPORTANT: Export and commit the configuration now! (`drush cex, git add config/sync, git commit -m "Exported config."`, git push)
* Create the second static environment: `git checkout -b develop, git push -u origin develop`
* Create feature branches and push them
* Check out the screencast: https://www.youtube.com/watch?v=V7DmIR3NjLk
