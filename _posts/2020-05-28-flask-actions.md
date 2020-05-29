---
title: "Continious deployment to EC2 with Github Actions"
description: "Building on the last post, we take it one step further and use Github
Actions to automate everything. Just push and deploy."
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [linux,aws,ansible,github]
---

In my [previous
post](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html) I covered
how to setup an AWS EC2 instance with a barebones Flask project. At the end of the post
we automated this by using Ansible. In this post, we take it one step further and let
Github use Ansible to provision and configure our EC2 instance. If you haven't read the
last post, I recommend at least skimming through it to get an idea of what we're setting
up here.

Briefly, we want gunicorn running our flask server on port 8080. Then we want nginx to
use a reverse proxy to serve that up to port 80 so people can visit our website like
normal (or we can ping our REST API endpoints on port 80). We want to use Ansible via
Github Actions to provision and configure everything in our EC2 instance automatically
so that whenever we push the master branch to Github it updates our deployment.

This is not intended as a large-scale solution, but if you're running a small website or
REST API that you would like to demo to people, this streamlines some of the process of
continously deploying.

## Setup

### Generate repository

First, generate a new repository based on my template by clicking
[here](https://github.com/wesbarnett/flask-project/generate). The contents of the flask
repository are discuess in the previous post
[here](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html#flask-project-setup).

Go ahead and clone your repository to your local machine so you can make changes to the
Flask server later.

To run the gunicorn server locally for testing, in it's current state you need at least
the `gunicorn` and `flask` packages. At the top level of the respository run:

```bash
gunicorn --chdir application -b :8080 app:app
```

The Flask application only has one route at `/` that simply prints "It works!". If
running locally you can go to <a href="http://localhost:8080">localhost:8080</a> to see
the default route.

### Generate SSH keypair

Create an SSH keypair to be used between Github and AWS. Personally I prefer to do this
locally on my machine using `ssh-keypair`, but you can use any third party tool. Don't
include a passphrase.

### Copy private key to Github

Next you will need to copy and paste the *private* key into the secrets for your new
repository that you generated. To do that, go to the settings for your repository and
then go to "Secrets" on the left hand menu. Click "New Secret" and name your new secrets
`AWS_EC2_KEY`. It must be named this since the value will be used in our Github Actions.
Paste in the private key and save. The private key should begin with (`OPENSSH` might
just be `SSH` in your implementation):

```
-----BEGIN OPENSSH PRIVATE KEY-----
```

{% include note.html content="Your private key is encrypted before it reaches Github and
not decrypted until it is used in a workflow <a
href='https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets'>according
to Github</a>. Still, understand the security implications of sharing your private key
with a third-party."
%}

![]({{ site.baseurl }}/images/github_actions/new_secret.png "Paste in your SSH
private key here.")

### Copy public key to AWS EC2

Next, copy the public key and import it into AWS. Go
[here](https://console.aws.amazon.com/ec2/#KeyPairs:) and select the white "Actions"
dropdown box. Click import key, name your key, and then either browse for the public key
or paste it in. It doesn't matter what you name your key here as long as you know it's
associated with your Github.

![]({{ site.baseurl }}/images/github_actions/aws_public.png "Paste in your SSH
public key here.")

### Create EC2 instance and associate Elastic IP

After this create an EC2 instance and associate an Elastic IP address with this. If you
don't know how to do this, you can follow the section in my previous post
[here](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html#aws-setup).
Return here after you have created an instance and associated an Elastic IP with it.

{% include warning.html content="If you're just testing this out and not wanting to run
your Flask site yet, be sure to stop your EC2 instance when you are finished testing."
%}

### Create an A record for domain

Create an A record for your domain or subdomain for the IP address. I personally use
[namecheap.com](https://namecheap.com). To add an A record for your domain or subdomain,
follow [this
article](https://www.namecheap.com/support/knowledgebase/article.aspx/319/2237/how-can-i-set-up-an-a-address-record-for-my-domain).

### Update Ansible configuration

Lastly, in your repository update `ansible/deploy/hosts` for your own domain. In the
file replace every instance of `test.barnett.science` with your domain name. You can
also change the `app_name` from `flask-project` to whatever you desire, but it is not
necessary. This is used as the systemd unit name that runs gunicorn as well as the
directory of where the repository will be cloned.

{% include hc.html header="ansible/deploy/hosts" body="
[webservers]
test.barnett.science

[webservers:vars]
domain_name=test.barnett.science
app_name=flask-project
ansible_ssh_user=ubuntu
" %}

Commit the change and push to your Github repository. You should now be able to visit
your domain and see the text "It works!".

## Usage

Now that you have it working, simply make updates to your code and push. It's
recommended you create another branch to for development and only merge to master when
it is ready for production. Anything pushed to the master branch will be automatically
deployed to your EC2 instance, so be sure to test locally! When confident with the
changes, merge with master and push.

If you make updates to the directory structure, like moving `__init__.py` to another
location, you may break the Github action from working. It expects
`__init__.py` to be under `application/app`. If you want it elsewhere, you'll need to
modify the ansible templates yourself. Other than that, there shouldn't be any
restrictions on what you can change.

You can add additional Python packages to the top-level `requirements.txt` and they will
be installed automatically as part of the ansible provisioning.

To learn more about how this repository is setup, see [this
section](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html#flask-project-setup).
