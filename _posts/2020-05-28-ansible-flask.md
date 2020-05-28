---
title: "Run a Flask app on AWS EC2"
description: "A guide for running a simple flask app on AWS EC2 along with a simple
Ansible playbook."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [linux,aws,ansible]
---

Let's say you've created a [Flask](https://flask.palletsprojects.com/en/1.1.x/)
application and you're ready to launch it to AWS. At this point you may not care about
scaling yet, and you just want to get it in production so you can demo it to prospective
clients or employers. How do you get that up and running quickly?

In this blog post we'll set up a simple Flask project to be run on an AWS EC2 instance
using [gunicorn](https://gunicorn.org/) and [nginx](https://nginx.org/).

After walking through this manually, we'll automate the setup using
[Ansible](https://www.ansible.com/)

## Flask project setup

I'm assuming you have already created your Flask application and know a little bit about
how to run it locally. I have a barebones Flask project located
[here](https://github.com/wesbarnett/flask-project) that I am using as a model for this
blog post. All of the routes/views are located in `application/app/routes.py` in the
project.

If you end up using your own Flask project, be sure that the directory structure is the
same in order to follow this post. Specifically, `__init__.py` should be in `application/app` and
should have contents that indicate where the Flask routes are stored. For example, in
the skeleton used above they are in `routes.py`, so the contents of `__init__.py` is:

```python
from flask import Flask
app = Flask(__name__)
from app import routes
```

Here's the directory structure:

```
application/
    app/
        __init__.py
        routes.py
requirements.txt
```

### Running locally

To run the repository locally, clone it to your machine:

```bash
git clone https://github.com/wesbarnett/flask-project
```

Then install the requirements. In this example I'm creating a virtual environment:
```bash
cd flask-project
python3 -m venv venv
source ./venv/bin/activate
python -m pip install flask gunicorn
```

Then to run locally do:
```bash
gunicorn --chdir application -b :8080 app:app
```

Then visit [http://localhost:8080](http://localhost:8080) and you should see a line that
says "It works!".


## AWS Setup

### Create and launch EC2 instance

So now you have your Flask project and you're ready to move it to an AWS EC2 instance.
It's time to login to your [AWS console's EC2
dashboard](https://console.aws.amazon.com/ec2). From there click the big orange "Launch
Instance" button.

![]({{ site.baseurl }}/images/aws/aws_ec2_dash.png "AWS EC2 Dashboard")

Let's use an Ubuntu Server 18.04 LTS image, so select that on the
next page. From there you can choose what instance type you want. If you are able,
choose the free tier eligible one if just testing this out. 

![]({{ site.baseurl }}/images/aws/aws_ubuntu.png "AWS EC2 image selection")

After selecting what instance type go ahead and skip ahead to step 6 where you can
configure your security group. Add a rule for port 80 and possibly 443 if you're going
to use HTTPS later, or choose a security group that already has those ports open. Simply
choose "HTTP" under type and it will automatically ensure port 80 is open for all
traffic.  Otherwise, you won't be able to access your application in a web browser.

![]({{ site.baseurl }}/images/aws/aws_security.png "AWS EC2 security group")

When you click "Review and launch" you will get a warning that you are allowing traffic
from anywhere. Then click "Launch". At this point you'll be asked to choose or create a
keypair that you will use when you ssh into your instance.

{% include warning.html content="Ensure you stop the instance when you are done!" %}

### Allocate elastic IP address

Next we want an elastic IP address. The advantage of this is that if we bring an
instance down and bring another one up, we can just move the IP address to be associated
with that new instance. Additionally if you stop an instance and restart it, without an
elastic IP address, you will have to find the new IP address of your instance.

![]({{ site.baseurl }}/images/aws/aws_elastic_ip.png "AWS Elastic IP allocation")

Under "Network & Security" on the left hand menu click "Elastic IPs". Then click
the orange "Allocate Elastic IP Address" button on the top right. Leave the settings
alone and click the orange "Allocate" button.

At the top you'll see a green bar with a button that says "Associate this Elastic IP
Address". Click that. Now choose your running instance and associate the IP address with
that instance.


## Domain name

If you are going to use a domain or subdomain, go ahead create an A record for your domain using
your elastic IP address. Personally I have been using
[namecheap.com](https://namecheap.com) as my registrar. [Here is an
article](https://www.namecheap.com/support/knowledgebase/article.aspx/319/2237/how-can-i-set-up-an-a-address-record-for-my-domain)
on how to add an A address record for namecheap.

## Setup instance

Once your instance is running you'll need to ssh into the instance. From a Linux machine
this looks like this:

```bash
ssh -i ~/.ssh/aws.pem ubuntu@<ip-address>
```

Here `~/.ssh/aws.pem` is my key that I associated with this instance. The IP address is
the Elastic IP you allocated and associated above.

Now you'll want to clone your Flask project to the instance. For my barebones project it
would be:

```bash
git clone https://github.com/wesbarnett/flask-project
```

### Install packages

We won't be using a virtual environment on this instance. You'll need to install a few
Ubuntu packages next:

```bash
sudo apt-get update
sudo apt-get install nginx gunicorn3 python3-pip python3-flask
```

For the barebones project this is all we need.

### Test gunicorn

Now you should be able to test `gunicorn`. Note that the executable here is named
`gunicorn3`. This runs the server in the background, makes a GET request to test it, and
then puts the server back into foreground. You should see "It works!" when you run curl:


```bash
gunicorn3 --chdir application -b :8080 app:app &
curl http://localhost:8080
fg
```

### gunicorn systemd unit

We don't want to have to type in `gunicorn3` manually to run our Flask server. Let's
have this occur automatically at boot by creating and enabling a systemd unit. Create a
file with the following contents at the indicated location:

{% include hc.html header="/etc/systemd/system/gunicorn.conf" body="
[Unit]
Description=gunicorn to serve flask-project
After=network.target

[Service]
WorkingDirectory=/home/ubuntu/flask-project
ExecStart=/usr/bin/gunicorn3 -b 0.0.0.0:8080 --chdir application app:app

[Install]
WantedBy=multi-user.target" %}

Change `flask-project` to the name of your project in the `WorkingDirectory` above.

Now to run your gunicorn service do:

    sudo systemctl start gunicorn

You should now be able to do `curl http://localhost:8080` and get a message that says
"It works!".

To enable this to run on boot, do:

    sudo systemctl enable gunicorn

If you ever want to restart the server, do:

    sudo systemctl restart gunicorn

### nginx configuration

We don't want to expose gunicorn to the web directly. It's not designed for taking
direct traffic and [susceptible to denial-of service attacks if you do
that](https://docs.gunicorn.org/en/stable/deploy.html). Instead we'll use
[nginx](https://nginx.org/) to take the traffic and setup a reverse proxy to gunicorn.

First remove the default configuration from being enabled:

    sudo rm /etc/nginx/sites-enabled/default

Then create a new configuration file:

{% include hc.html header="/etc/nginx/sites-available/flask-project.conf" body="
server {
    listen 80;
    listen [::]:80;

    location / {
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host              $http_host;
        proxy_pass http://localhost:8080;
    }
}" %}

Now enable it by creating a symlink:

    sudo ln -s /etc/nginx/sites-available/flask-project.conf /etc/nginx/sites-enabled/

Finally, start nginx:

    sudo systemctl start nginx

You should now be able to enter your Elastic IP address and domain name into a web
browser and see your website! 

To enable nginx on boot do:

    sudo systemctl enable nginx

Whenever you make a change to your code, you'll need to pull it in with git and restart
the gunicorn service.

## Automate setup with Ansible

Let's make setting up our server a even easier using Ansible. For this section I
recommend creating a new, clean instance. If you're not using the previous instance from
above simply terminate it and release it's elastic IP address. You can then associate
the Elastic IP address with this new instance if you want or create a new one. Simply go
back and follow the AWS setup steps again.

Install [Ansible](https://www.ansible.com/) to your local machine. Ansible is a remote
configuration and provisioning tool. For Linux distributions simply use your package
manager. For MacOS you can use Homebrew. I'm not
sure what options are available for Windows. You don't need to install anything on your
AWS instance, not even Ansible!

### Create playbook

Ansible consists of playbooks that you use to tell it what to do. These playbooks then
refer to configuration file templates. On your local machine create a new directory to
contain your playbook and templates:

    mkdir aws-flask-playbook
    cd aws-flask-playbook

In this directory create an Ansible configuration file:

{% include hc.html header="ansible.cfg" body="
[defaults]
host_key_checking = False
inventory         = ./deploy/hosts" %}

Now create a directory named `deploy` and in it a file named `hosts`. Add your Elastic
IP address under `[webservers]`. You also need to change the location of your private
key file. `github_user` and `app_name` are used in the Github url as well as naming some
of the files later.

{% include hc.html header="deploy/hosts" body="
[webservers]
<i>your-elastic-ip</i>

[webservers:vars]
ansible_ssh_user=ubuntu
ansible_ssh_private_key_file=/home/wes/.ssh/aws.pem
github_user=wesbarnett
app_name=flask-project
domain_name=test.barnett.science
" %}

Then create a playbook:

<pre style="margin-bottom: 0 !important; border-bottom:none;
           padding-bottom:0.8em;">deploy.yaml</pre>
<pre style="margin-top: 0; border-top-style:dashed; padding-top:
            0.8em;"># Ansible playbook for deploying a Flask app
{% raw %}
---
- hosts: webservers
  become: yes
  become_method: sudo
  tasks:
  - name: install packages
    apt: 
      name: "{{ packages }}"
      update_cache: yes
    vars:
      packages:
        - nginx
        - gunicorn3
        - python3-pip

- hosts: webservers
  become: yes
  become_method: sudo
  tasks:
  - name: clone repo
    git:
      repo: 'https://github.com/{{ github_user }}/{{ app_name }}.git'
      dest: /srv/www/{{ app_name }}
      update: yes

- hosts: webservers
  become: yes
  become_method: sudo
  tasks:
  - name: Install needed python packages
    pip:
      requirements: requirements.txt
      chdir: /srv/www/{{ app_name }}

- hosts: webservers
  become: yes
  become_method: sudo
  tasks:
  - name: template systemd service config
    template:
      src: deploy/gunicorn.service
      dest: /etc/systemd/system/{{ app_name }}.service
  - name: start systemd app service
    systemd: name={{ app_name }}.service state=restarted enabled=yes
  - name: template nginx site config
    template:
      src: deploy/nginx.conf
      dest: /etc/nginx/sites-available/{{ app_name }}.conf
  - name: remove default nginx site config
    file: path=/etc/nginx/sites-enabled/default state=absent
  - name: enable nginx site
    file:
      src: /etc/nginx/sites-available/{{ app_name }}.conf
      dest: /etc/nginx/sites-enabled/{{ app_name }}.conf
      state: link
      force: yes
  - name: restart nginx
    systemd: name=nginx state=restarted enabled=yes
{% endraw %}</pre>

The playbook does the following:

1. Installs `nginx`, `gunicorn3`, and `python-pip`.
2. Clones the git repository `https://github.com/<github_user>/<app_name>` to `/srv/www/{{ app_name }}`.
3. Installs the python packages listed in `requirements.txt` of the repo to be cloned.
   `flask` should be listed if you are running a flask server.
4. Installs and starts a systemd service to run gunicorn. 
5. Installs and enables an `nginx` configuration that forwards the gunicorn web service
   to port 80.

After creating create the two configuration file templates that the playbook uses in the
`deploy` directory. These are template files - you do not and should not change anything
in them, since Ansible will automatically fill in the variables from `deploy/hosts`.

<pre style="margin-bottom: 0 !important; border-bottom:none;
           padding-bottom:0.8em;">deploy/nginx.conf</pre>
<pre style="margin-top: 0; border-top-style:dashed; padding-top:
            0.8em;">{% raw %}
server {
    listen 80;
    listen [::]:80;

    server_name {{ domain_name }};
    location / {
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host              $http_host;
        proxy_pass http://localhost:8080;
    }
}{% endraw %}</pre>

<pre style="margin-bottom: 0 !important; border-bottom:none;
           padding-bottom:0.8em;">deploy/gunicorn.service</pre>
<pre style="margin-top: 0; border-top-style:dashed; padding-top:
            0.8em;">
{% raw %}
[Unit]
Description=gunicorn to serve {{ app_name }}
After=network.target

[Service]
WorkingDirectory=/srv/www/{{ app_name }}
ExecStart=/usr/bin/gunicorn3 -b 0.0.0.0:8080 --chdir application app:app

[Install]
WantedBy=multi-user.target
{% endraw %}</pre>

### Run playbook

After setting everything up above, you can now run the playbook.

    ansible-playbook deploy.yaml

![]({{ site.baseurl }}/images/aws/aws_ansible_output.png "Output after running the
playbook")

After running the playbook you should be able to visit your Elastic IP address in a web
browser and see "It works!" for the barebones Flask project.

Now as you make changes, simply push to Github as usual. When you're ready to update
your server, just re-run the playbook.

{% include warning.html content="If you're just testing this out and not wanting to run
your Flask site yet, be sure to stop or terminate your EC2 instance." %}
