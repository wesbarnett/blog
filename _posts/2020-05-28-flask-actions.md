---
title: "Continuous deployment of a Flask server with Github Actions"
description: "Use a Github Workflow and Action to continuously deploy a Flask server to
an EC2 instance, or any other Linux server."
layout: post
toc: false
comments: true
image: images/github_actions/diagram.png
hide: false
search_exclude: false
categories: [linux,aws,ansible,github]
---

In my [previous
post](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html) I covered
how to setup an AWS EC2 instance with a bare bones Flask project. At the end of the post
we automated this by using Ansible. In this post, we take it one step further and let
Github use Ansible to provision and configure our EC2 instance. If you haven't read the
last post, I recommend at least skimming through it to get an idea of what we're setting
up here.

{% include tip.html content="You can actually use this Github Workflow and Action with
any cloud provider or your own personal Linux server. You just need to ensure ports 80
and 443 are open and copy the public key to the remote server. Additionally you may need
to change the ansible user."%}

We want to use an [Ansible playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) via Github
Actions to provision and configure everything in our EC2 instance automatically so that
whenever we push the master branch to Github it updates our deployment. The Ansible
playbook, [found
here](https://github.com/wesbarnett/flask-project/blob/master/ansible/deploy.yaml),
installs the Ubuntu packages we need (pip, gunicorn, nginx); clones our Github
repository and installs anything in `requirements.txt`; creates a systemd unit to run
gunicorn and starts it; and creates an nginx configuration and starts it. The nginx
configuration is just a reverse proxy from `http://localhost:8080`&mdash;where gunicorn is
serving our Flask server&mdash;to port 80, the standard web server port.

![]({{ site.baseurl }}/images/github_actions/diagram.png "We'll utilize a Github Workflow
that consists of an Action that uses Ansible to configure our AWS EC2 instance to serve
our Flask server.")

This is not intended as a large-scale solution, but if you're running a small website or
REST API that you would like to demo to people, this streamlines some of the process of
continuously deploying.

## Setup

### Generate repository

First, generate a new repository based on my template by clicking
[here](https://github.com/wesbarnett/flask-project/generate). The contents of the flask
repository are discussed in the previous post
[here](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html#flask-project-setup).

Go ahead and clone your repository to your local machine so you can make changes to the
Flask server later.

To run the gunicorn server locally for testing, in it's current state you need at least
the `gunicorn` and `flask` packages. At the top level of the repository run:

```bash
gunicorn --chdir application -b :8080 app:app
```

The Flask application only has one route at `/` that simply prints "It works!". If
running locally you can go to <a href="http://localhost:8080">localhost:8080</a> to see
the default route.

{% include tip.html content="After you generate a new repository from the template, an
issue will be opened in the new repository also outlining these steps."%}

### Generate SSH key pair

Create an SSH key pair to be used between Github and AWS. Personally I prefer to do this
locally on my machine using `ssh-keypair`, but you can use any third party tool. Don't
include a passphrase.

#### Copy private key to Github

Next you will need to copy and paste the *private* key into the secrets for your new
repository that you generated. To do that, go to the settings for your repository and
then go to "Secrets" on the left hand menu. Click "New Secret" and name your new secrets
`AWS_EC2_KEY`. It must be named this since the value will be used in our Github Actions.
Paste in the private key and save. The private key should begin with (`OPENSSH` might
be `SSH` or `RSA` in your implementation):

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

#### Copy public key to AWS EC2

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

#### Create an A record for domain

Create an A record for your domain or subdomain for the IP address. I personally use
[namecheap.com](https://namecheap.com). To add an A record for your domain or subdomain,
follow [this
article](https://www.namecheap.com/support/knowledgebase/article.aspx/319/2237/how-can-i-set-up-an-a-address-record-for-my-domain). It may take a few minutes for DNS servers to be updated.

### Update Ansible configuration

Lastly, in your repository update `ansible/deploy/hosts` for your own domain. In the
file replace the instance of `test.barnett.science` with your domain name (you could
actually list several domain names if you wanted to deploy to several different
servers). Note that although Ansible allows IP addresses, the templates for this project
expect this to be a domain name. You can also change the `app_name` from `flask-project`
to whatever you desire, but it is not necessary. This is used as the systemd unit name
that runs gunicorn as well as the directory of where the repository will be cloned.

{% include hc.html header="ansible/deploy/hosts" body="
[webservers]
test.barnett.science

[webservers:vars]
app_name=flask-project
ansible_ssh_user=ubuntu
SSL=False
" %}

Commit the change and push to your Github repository. You should now be able to visit
your domain and see the text "It works!".

### SSL

This step is optional for those who want to serve using HTTPS. Ensure that your site is
served over HTTP before proceeding.

{% include note.html content="Although it is very simple to enable HTTPS, you can run
into problems if trying to switch back to HTTP-only later. Specifically, web browsers
that had been accessing the site via HTTPS will possibly now see the site as insecure
and may not be able to access content. In other words, if choosing to serve over HTTPS,
stick with it."%}

To enable SSL for your site, ensure that port 443 is open in your EC2 security group for
inbound connections. Then SSH into your EC2 instance and follow the instructions to use
Let's Encrypt's certbot [found
here](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx).

After this is complete, simply change `SSL=True` in `ansible/deploy/hosts` and push.
This sets a boolean variable such that the nginx configuration for your domain is
changed to serve using HTTPS. Additionally all HTTP traffic will be redirected to HTTPS.
This is an important step, because if you do not change this variable the nginx
configuration will be overwritten with one that only serves over HTTP the next time you
push.

## Next steps

Now that you have everything working, simply make updates to your code and push. It's
recommended you create another branch for development and only merge to master when
it is ready for production. Anything pushed to the master branch will be automatically
deployed to your EC2 instance, so be sure to test locally! When confident with the
changes, merge with master and push.

If you make updates to the directory structure, like moving `__init__.py` to another
location, you may break the Github action from working. It expects
`__init__.py` to be under `application/app`. If you want it elsewhere, you'll need to
modify the Ansible templates yourself. Other than that, there shouldn't be any
restrictions on what you can change or add to your flask server (templates, static
files, etc.).

You can add additional Python packages to the top-level `requirements.txt` and they will
be installed automatically as part of the Ansible provisioning.

To learn more about how this repository is setup, see [this
section](https://barnett.science/linux/aws/ansible/2020/05/28/ansible-flask.html#flask-project-setup).

Lastly, you can always check how the Github action performed by going to the "Actions"
tab in your new repository. Additionally you can still ssh into your EC2 instance and
check the status of both the gunicorn service (`flask-project.service` by default) and
nginx service (`nginx.service`) using systemd. Note that if you make any changes to
`flask-project.service` or the nginx configuration you should do that in the ansible
template in your repository, not in your instance directly; otherwise, any changes you
make to those files will be overwritten by your ansible playbook when you push later.
