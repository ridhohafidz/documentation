# Gitlab CI/CD for Django
> Disclaimer: This article doesn't bring anything new to the table. It is but a mere compilation of several article(s) and stackoverflow answer(s) in a single place

> Words to go by: If an app can be deployed manually, it most probably can be automated.

## Table of Contents
- [Gitlab CI/CD for Django](#gitlab-cicd-for-django)
  - [Table of Contents](#table-of-contents)
  - [Requirements](#requirements)
  - [Configuration Setup](#configuration-setup)
  - [Running the App](#running-the-app)
  - [Gunicorn Configuration](#gunicorn-configuration)
  - [Nginx Configuration](#nginx-configuration)
  - [Create YAML file](#create-yaml-file)
  - [Check the Pipeline](#check-the-pipeline)
  - [References](#references)

## Requirements

1. Python packages and PostgreSQL
    To begin the process, we’ll download and install all of the items we need from the Ubuntu repositories (which thankfully also works for Debian distro). We will use the Python package manager `pip` to install additional components a bit later.

    We need to update the local `apt` package index and then download and install the packages. The packages we install depend on which version of Python our project will use.

    If you are using **Python 2**, type:

    ```bash
    $ sudo apt-get update
    $ sudo apt-get install python-pip python-dev libpq-dev postgresql postgresql-contrib nginx
    ```

    If you are using Django with **Python 3**, type:

    ```bash
    $ sudo apt-get update
    $ sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx
    ```

2. Database access and information
    Assuming you have the database already up and running on server `MyDatabaseServer`, you will only need to install Postgres client to connect to the databases on that server. The key here is to modify `pg_hba.conf` on `MyDatabaseServer` to enable it to be 'listened to' by server `MyAppServer`.

    Don't forget to obtain necessary information such as **database name**, **user**, and **password** for the connection to work.

## Configuration Setup

1. Design directory structure
    Before going any further, this is the best time to consult the developers regarding the development phases they have in mind.
    Usually, they are `development`, `staging`, and `production`. So it will be best to make three separate directories for each project phase.
    So for example, if you're working on a project called `my-awesome-project`, you'd probably want to put all you project files under `my-awesome-project` directory

        .
        ..
        |-- my-awesome-project
            |-- dev
                |-- myproject
            |-- staging
                |-- myproject
            |-- prod
                |-- myproject

    Knowing the development phases helps us avoid redoing some steps as a result of unplanned directories structures.

    Furthermore, it is also advisable to create your user name for CI/CD purposes the same as the project name instead of the `gitlab` most tutorials use. It adds to clearer directory structure as there's no sudden `gitlab` in your path, keeping it clean from any platform-biased term.

2. Create Python virtual environment

    Quoting from [this tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04):

    > Installing Django into an environment specific to your project will allow your projects and their requirements to be handled separately.

    Following that advice, we will be installing Django within a virtual environment. In fact, we will be installing all our Python requirements within a virtual environment for easier management.

    To do this, we first need access to the `virtualenv` command. This can be installed with `pip`.

    If you are using **Python 2**, upgrade `pip` and install the package by typing:

    ```bash
    $ sudo -H pip install --upgrade pip
    $ sudo -H pip install virtualenv
    ```

    If you are using **Python 3**, upgrade `pip` and install the package by typing:

    ```bash
    $ sudo -H pip3 install --upgrade pip
    $ sudo -H pip3 install virtualenv
    ```

    With `virtualenv` installed, we can start setting our project up.

    Move into your Django project directory, e.g. `~/dev/myproject` , and from within that, create a Python virtual environment by typing:

    ```bash
    $ cd ~/dev/myproject
    $ virtualenv myprojectenv
    ```

    This will create a directory called `myprojectenv` within your `myproject` directory. Inside, it will install a local version of Python and a local version of `pip`. We can use this to install and configure an isolated Python environment for each of our projects.

3. Install project's requirements

   Before we install our project’s Python requirements, we need to activate the virtual environment that we've created in the previous step. You can do that by typing:

    ```bash
    source myprojectenv/bin/activate
    ```

    Your prompt should change to indicate that you are now operating within a Python virtual environment. It will look something like this: `(myprojectenv) user@host:~/dev/myproject$`.

    With your virtual environment active, install Django, Gunicorn, and the psycopg2 PostgreSQL adaptor with the local instance of `pip`:

    >>>
    Note:
    Regardless of which version of Python you are using, when the virtual environment is activated, you should use the pip command (not pip3).

    ```bash
    (myprojectenv) $ pip install django gunicorn psycopg2
    ```

    Then, you may want to install the project's own dependencies listed in `requirements.txt`:

    ```bash
    (myprojectenv) $ pip install -r requirements.txt
    ```

    You should now have all of the software needed to run your Django project.

4. Adjust the project's settings

    a. Allowed hosts

      Adjust the settings in `~/dev/myproject/myproject/settings.py`. Start with modifying `ALLOWED_HOSTS` to include `MyAppServer` IP address.

    ```bash
    (myprojectenv) $ nano ~/myproject/myproject/settings.py
    . . .
    # The simplest case: just add the domain name(s) and IP addresses of your Django server
    # ALLOWED_HOSTS = [ 'example.com', '203.0.113.5']
    # To respond to 'example.com' and any subdomains, start the domain with a dot
    # ALLOWED_HOSTS = ['.example.com', '203.0.113.5']
    ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP', . . .]
    ```

    b. Database settings

    The databases settings and other confidential settings should not be plainly written in the `settings.py`.
    Instead, it's generally a good idea to move it out the main `settings.py` file.
    There are many ways to store and retrieve environment variable in a Django project. A bunch of great example can be found from [this article I happen to stumble on](https://code.djangoproject.com/wiki/SplitSettings).

## Running the App

With all the settings all set, let's try running the app on `port 7030` (or any free port you want) by running these Django deployment sequence:

1. `(myprojectenv) $ ~/dev/myproject/manage.py makemigrations`
2. `(myprojectenv) $ ~/dev/myproject/manage.py migrate`
3. `(myprojectenv) $ ~/dev/myproject/manage.py collectstatic`
4. `(myprojectenv) $ ~/dev/myproject/manage.py runserver 0.0.0.0:7030`

You should be able to see your project default page if you visit your `MyAppServer`'s domain name or IP address followed by `:7030`.

## Gunicorn Configuration

Now, we need to test Gunicorn to make sure it can serve the application. We can test this by going into the project's directory and using `gunicorn` to load the project's WSGI module:

```bash
(myprojectenv) $ cd ~/dev/myproject
(myprojectenv) $ gunicorn --bind 0.0.0.0:7030 myproject.wsgi
```

This will start Gunicorn on the same interface that the Django server was running on.

When you are finished testing, hit **CTRL-C** in the terminal window to stop Gunicorn.

We’re now finished configuring our Django application. We can back out of our virtual environment by typing:

```bash
(myprojectenv) $ deactivate
```

The virtual environment indicator in your prompt should now be removed.

We have tested that Gunicorn can interact with our Django application, but we should implement a more robust way of starting and stopping the application server. To accomplish this, we’ll make a `systemd` service file.

Normally, you only need to create a `systemd` service file for Gunicorn with the following content

```bash
$ sudo nano /etc/systemd/system/gunicorn.service
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=my-awesome-project
Group=www-data
WorkingDirectory=/home/my-awesome-project/myproject
ExecStart=/home/my-awesome-project/myproject/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/my-awesome-project/myproject/myproject.sock myproject.wsgi:application

[Install]
WantedBy=multi-user.target
```

But in our case, we need a little modification because we want the `Gunicorn` to be able to serve `development`, `staging`, and `production` app on the same server. It is desirable to be able to stop and start an individual `Gunicorn` instance and also consider them as a single service that can be stopped or started together. The `systemd` solution involves creating a **target** which will be used to treat all instances as a single service. Then, a single **template unit** will be used to allow us to add multiple Gunicorns to the **target**.

Here's an example of `/etc/systemd/system/gunicorn.target`:

```bash
$ sudo nano /etc/systemd/system/gunicorn.target
[Unit]
Description=Gunicorn
Documentation=https://example.com/path/to/your/docs

[Install]
WantedBy=multi-user.target
```

When enabled, that **target** will start at boot.

Now for the template file `/etc/systemd/systemd/gunicorn@.service`:

```bash
$ sudo nano /etc/systemd/system/gunicorn@.service
[Unit]
Description=gunicorn daemon
After=network.target
PartOf=gunicorn.target
# Since systemd 235 reloading target can pass through
ReloadPropagatedFrom=gunicorn.target

[Service]
User=my-awesome-project
Group=www-data
WorkingDirectory=~/home/my-awesome-project/%i/myproject

ExecStart=/home/my-awesome-project/%i/myproject/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/my-awesome-project/%i/myproject/myproject.sock myproject.wsgi:application

[Install]
WantedBy=gunicorn.target
```

To **start or stop all Gunicorn instances** at once, we can simply execute:

```bash
$ sudo systemctl start gunicorn.target
$ sudo systemctl stop gunicorn.target
```

We can use `enable` and `disable` to **add and remove new Gunicorn instances** on the fly:

```bash
$ systemctl enable gunicorn@dev
$ systemctl disable gunicorn@staging
```

We can also **stop and start a specific service** by itself:

```bash
$ sudo systemctl start gunicorn@dev
$ sudo systemctl stop gunicorn@dev
$ sudo systemctl restart gunicorn@dev
```

Check the **status of the process** to find out whether it was able to start by executing:

```bash
$ sudo systemctl status gunicorn
```

Next, check for the existence of the myproject.sock file within your project directory:

```bash
$ ls /home/my-awesome-project/dev/myproject
manage.py  myproject  myprojectenv  myproject.sock  static ...
```

If the `systemctl` status command indicated that an error occurred or if you do not find the `myproject.sock` file in the directory, it’s an indication that `Gunicorn` was not able to start correctly. Check the `Gunicorn` process logs by typing:

```bash
$ sudo journalctl -u gunicorn
```

Take a look at the messages in the logs to find out where `Gunicorn` ran into problems. There are many reasons that you may have run into problems, but often, if `Gunicorn` was unable to create the socket file, it is for one of these reasons:

- The project files are owned by the root user instead of a sudo user
- The `WorkingDirectory` path within the `/etc/systemd/system/gunicorn@.service` file does not point to the project directory
- The configuration options given to the `Gunicorn` process in the `ExecStart` directive are not correct. Check the following items:
  - The path to the `Gunicorn` binary points to the actual location of the binary within the virtual environment
  - The `--bind` directive defines a file to create within a directory that `Gunicorn` can access
  - The `myproject.wsgi:application` is an accurate path to the `WSGI` callable. This means that when you’re in the `WorkingDirectory`, you should be able to reach the callable named application by looking in the `myproject.wsgi` module (which translates to a file called `./myproject/wsgi.py`)

If you make changes to the `/etc/systemd/system/gunicorn.service` file, reload the daemon to re-read the service definition and restart the `Gunicorn` process by executing:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart gunicorn.target
```

Make sure you troubleshoot any of the above issues before continuing.

## Nginx Configuration

Now that `Gunicorn` is set up, we need to configure Nginx to pass traffic to the process.

Start by creating and opening a new server block in Nginx’s `sites-available` directory. Assuming we're setting up for the `development` app, we'll execute the following command:

```bash
$ sudo nano /etc/nginx/sites-available/myproject-dev
```

Inside, open up a new server block. , We will start by specifying that this block should listen on the `port 7030` and that it should respond to our server’s domain name or IP address:

```bash
$ sudo nano /etc/nginx/sites-available/myproject-dev
...
server {
    listen 7030;
    listen [::]:7030;
    server_name server_domain_or_IP;
}
...
```

Next, we will tell Nginx to ignore any problems with finding a favicon. We will also tell it where to find the static assets that we collected in our `~/dev/myproject/static` directory. All of these files have a standard URI prefix of `/static`, so we can create a location block to match those requests:

```bash
$ sudo nano /etc/nginx/sites-available/myproject-dev
...
server {
    listen 7030;
    listen [::]:7030;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/my-awesome-project/dev/myproject;
    }
}
...
```

Finally, we’ll create a location / {} block to match all other requests. Inside of this location, we’ll include the standard `proxy_params` file included with the Nginx installation and then we will pass the traffic to the socket that our `Gunicorn` process created:

```bash
$ sudo nano /etc/nginx/sites-available/myproject-dev
...
server {
    listen 7030;
    listen [::]:7030;
    server_name server_domain_or_IP;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/my-awesome-project/dev/myproject;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/my-awesome-project/dev/myproject/myproject.sock;
    }
}
...
```

Save and close the file when you are finished. Now, we can enable the file by linking it to the sites-enabled directory:

```bash
sudo ln -s /etc/nginx/sites-available/myproject-dev /etc/nginx/sites-enabled
```

Test your Nginx configuration for syntax errors by executing:

```bash
sudo nginx -t
```

If no errors are reported, go ahead and restart Nginx by typing:

```bash
sudo systemctl restart nginx
```

You should now be able to go to your server’s domain or IP address to view your application.

## Create YAML file

Now that all requirements are nice and set, and we know that to deploy a Django app all we need to do is run the following command in sequence:

1. **`~/path_to_app/manage.py makemigrations`**
2. **`~/path_to_app/manage.py migrate`**
3. **`~/path_to_app/manage.py collectstatic`**
4. **`~/path_to_app/manage.py runserver 0.0.0.0:[port]`**
5. **`gunicorn --bind 0.0.0.0:[port] myproject.wsgi`**

We can now start making the YAML file to automate the process.

1. Image

   ```yaml
   image: alpine:3.10
   ```

    Choosing the correct image is the first step to a successful script. It is advisable to find an image that includes necessary packages for your project to reduce the amount of things you need to install during the deployment process. When in doubt, go with the basic Linux distribution or anything you're familiar with. Using `:latest` may not produce the result you're expecting as it pulls random image according to where that tag is placed on the image, so **always specify the version of the image** you're using. It helps other people understand and keep your apps in a predictable environment for a better debugging experience.

2. Jobs and stages

    The jobs and stages need to be discussed with the developer beforehand. When there's no test to run, we can set up 3 jobs for each development phase with a single `deploy` stage.

    ```yaml
    image: alpine:3.10

    dev:
        stage: deploy
        ...

    staging:
        stage: deploy
        ...

    production:
        stage: deploy
        ...
    ```

3. Script

   The script is where we translate the whole manual deployment process into a series of `bash` commands. The customary first part of the script is usually SSH keys setup.

    ```yaml
    script:
        - apk add --no-cache rsync openssh
        - mkdir -p ~/.ssh
        - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
        - chmod 600 ~/.ssh/id_rsa
        - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    ```

    For the script to work, basically, you need an **SSH key-pair**. The private one is to be used by `gitlab` and the public one is to be put in the server as one of the `authorized_keys`.

    So first, you'll need to provide `gitlab` with a private SSH key. For security measure, **do not give your own private SSH key**. Create one that is used only by `gitlab`.

    ```bash
    # create user that will be used by gitlab
    $ adduser my-awesome-project

    # generate a RSA SSH key
    $ su -l my-awesome-project
    $ ssh-keygen
    ```

    The above commands can be run on any machine. But supposed you do it on `MyAppServer` server--the server you want your app to be deployed to--you can follow it up with the following commands:

    ```bash
    # authorize the key to log in using the public key and output the private one
    $ cd .ssh
    $ mv id_rsa.pub authorized_keys
    $ cat id_rsa && rm id_rsa
    ```

    Copy the private key shown and put it in a variable named `SSH_PRIVATE_KEY` in your project GitLab.

    If you do it on other server, you need to put the public key on `.ssh` directory on `MyAppServer` and rename it as `authorized_keys`.

    Now, we can follow up the rest of the script with deployment-related commands: *copying the files to the server*; *activates virtual environment*; *installing dependencies*; *deploy Django app*; *deactivate virtual environment*; *restart the Gunicorn server*.

    ```yaml
    script:
        - apk add --no-cache rsync openssh
        - mkdir -p ~/.ssh
        - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
        - chmod 600 ~/.ssh/id_rsa
        - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
        - rsync -rav --exclude '.git*' . my-awesome-project@server-a:~/dev/myproject/
        - ssh -tt my-awesome-project@server-a "cd ~/dev/myproject && source myprojectenv/bin/activate && pip --proxy http://your_company_proxy_server install -r requirements.txt && ~/dev/myproject/manage.py makemigrations && ~/dev/myproject/manage.py migrate && yes yes | ~/dev/myproject/manage.py collectstatic && deactivate"
        - ssh -tt my-awesome-project@server-a "sudo systemctl restart gunicorn@dev"
        - echo 'Finish deploying development updates'
    ```

4. Additional options

    We also want the above script to run only when there's changes in the development branch. Therefore, we need to add `except` in the `dev` job:

    ```yaml
    image: alpine:3.10

    dev:
        stage: deploy
        script:
            - apk add --no-cache rsync openssh
            - mkdir -p ~/.ssh
            - echo "$SSH_PRIVATE_KEY" >> ~/.ssh/id_rsa
            - chmod 600 ~/.ssh/id_rsa
            - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
            - rsync -rav --exclude '.git*' . my-awesome-project@server-a:~/dev/myproject/
            - ssh -tt my-awesome-project@server-a "cd ~/dev/myproject && source myprojectenv/bin/activate && pip --proxy http://your_company_proxy_server install -r requirements.txt && ~/dev/myproject/manage.py makemigrations && ~/dev/myproject/manage.py migrate && yes yes | ~/dev/myproject/manage.py collectstatic && deactivate"
            - ssh -tt my-awesome-project@server-a "sudo systemctl restart gunicorn@dev"
            - echo 'Finish deploying development updates'
        except:
            - master
            - staging
    ```

    Now, we only need to repeat those steps to create the script for `staging` and `production` jobs.

## Check the Pipeline

To check whether the YAML file actually does what you tell it to do, push some changes for each of the branch associated for each jobs you specify in the YAML: `dev`, `staging`, and `master`.

If all is well, you should see green check mark when the pipeline finishes. But if it's not, you could see what causes the error in the pipeline log and trace the error from there.

## References

- [Reference 1: https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04)
- [Reference 2: https://stackoverflow.com/questions/42219633/running-two-instances-of-gunicorn#42230370](https://stackoverflow.com/questions/42219633/running-two-instances-of-gunicorn#42230370)
- [Reference 3: https://code.djangoproject.com/wiki/SplitSettings](https://code.djangoproject.com/wiki/SplitSettings)
- [Reference 4: https://about.gitlab.com/blog/2017/09/12/vuejs-app-gitlab/](https://about.gitlab.com/blog/2017/09/12/vuejs-app-gitlab/)

---

**Scribble section**

If you manage to reach this part but never ask about the elephant in the room:

*Why Django, Postgres, Nginx, and Gunicorn?*

Well, I've been tasked to set up GitLab CI/CD for the app and it just so happens that the app is build with Django and Postgres. The Nginx and Gunicorn bit just happens to be there because [the only tutorial I manage to understand](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04)
is using those techs.
Fortunately for me, it meets the requirements and all parties involved are happy.

The bottom line is, one should always question techs choices when given the chance. There's always a chance that different combination of techs could produce better app.

Cyao!

---
