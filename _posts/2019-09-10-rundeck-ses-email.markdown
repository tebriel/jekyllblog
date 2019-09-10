---
layout: post
title:  "Rundeck Email using AWS SES"
date:   2019-09-10 15:34:00 -0400
categories: devops rundeck aws
---

## Overview

At [Pindrop][pindrop] we use [Rundeck][rundeck] to manage a lot of our operational toil. Rundeck has the
option to send email when jobs are complete. We have Amazon's [Simple Email Service][aws ses] configured to
send email as a trusted authority from Pindrop email accounts. Getting this configured in Rundeck was
non-obvious, so I wanted to write a small article about what it takes to get this working.

## Requirements

First you must configure [AWS SES][aws ses] properly. That's outside of the scope of this post, but generally
you'll need to set up some DNS records that prove you own the domain, and request a service limit increase
from AWS Support to be able to actually send out email.

You'll need to get a username/password using the `Create My SMTP Credentials` button in the AWS SES console
under the `SMTP Settings` tab.

Then you'll need to configure Rundeck, which will probably require some changes to get things working.

## Configuring Rundeck

Rundeck configuration `/etc/rundeck/rundeck-config.conf` is by default a conf file. This is all well and good
until you need to do something fancy, like configure advanced SMTP stuff. To do this you'll need to convert
your config file to a groovy config file.

### Groovy Config

There are a couple of differences between the `groovy` and the `conf` file format. The foremost is that you
must quote all strings on the right side of the `=`. This means you'll do the following, which is pretty
simple:


#### Original

```conf
loglevel.default = INFO
```

#### Groovy

```groovy
loglevel.default = "INFO"
```

Additionally, if you have any arrays you're indexing, you will need to quote them, again, an example is the
best help here. Note that both the `1` and the assigned values are quoted now.

#### Original

```conf
rundeck.storage.provider.1.type = db
rundeck.storage.provider.1.path = /keys
```

#### Groovy

```groovy
rundeck.storage.provider."1".type = "db"
rundeck.storage.provider."1".path = "/keys"
```

You'll also need to set up rundeck to use your new groovy config file. To do this, edit
`/etc/default/rundeckd` and add `export RDECK_CONFIG_FILE=/etc/rundeck/rundeck-config.groovy`.

Start up Rundeck and make sure it's happy with the configuration.


### Configuring SES (via Grails) in Rundeck

Rundeck uses [Grails][grails] and conveniently uses the grail configuration for email sending too. Now that
we've switched to a `groovy` format for our config file, we can add a new `grails` block. Use the following
template and put it in your `rundeck-config.groovy` (replacing values as appropriate, like
username/password/source address/aws region).

```grails
grails {
    mail {
        "default" {
            from = "rundeck@your_organization.com"
        }
        host = "email-smtp.us-east-1.amazonaws.com"
        port = 587
        username = "${SMTP_USERNAME}"
        password = "${SMTP_PASSWORD}"
        props = ["mail.smtp.starttls.enable":"true",
                 "mail.smtp.port":"587"]
    }
}
```

Finally restart Rundeck and create a job to send a test email, hopefully you're good to go!


[pindrop]: https://www.pindrop.com
[rundeck]: https://www.rundeck.com/
[aws ses]: https://aws.amazon.com/ses/
[grails]: https://grails.org/
