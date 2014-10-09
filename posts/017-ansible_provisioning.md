title: Setting up your own network infrastructure with Ansible
published: no

Writing code is easy, deploying them is hard. This is why many developers opt
for services such as Heroku and Google App Engine. However, this means that we
give up control to these providers, not to mention the hundreds of dollars
that we need to pay each month to keep our code running.

Over the last couple of weeks, I've started converting all of my server
instances to be managed by [ansible][ansible], a provisioning tool similar to
chef and puppet. The primary motivation for choosing ansible is that it plays
well with Python if required, and it does not require an additional server that
stores all the configurations. It simply uses ssh from your laptop.

[ansible]: http://www.ansible.com/

In this post, I'll go over the basics of what I have done, and how you can do
something similar with your own server setup.

-------------------------------------------------------------------------------

