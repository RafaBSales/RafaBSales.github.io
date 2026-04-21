---
# https://chirpy.cotes.page/posts/write-a-new-post/
layout: post
title: "Free Software Development - Report #5"
date: 2026-04-16 10:41 -0300
categories: [University of São Paulo, Linux]
tags: [git, email, patches]
toc: true
comments: false
media_subpath: /assets/img

description: Sending patches by email with git and a USP email
---

This report will follow my experience through the tutorials that can be found at
- [https://flusp.ime.usp.br/git/sending-patches-by-email-with-git/](https://flusp.ime.usp.br/git/sending-patches-by-email-with-git/), and
- [https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/](https://flusp.ime.usp.br/git/sending-patches-with-git-and-a-usp-email/).

## Sending patches by email with git

This tutorial focuses on the initial configuration and process of sending patches to the Linux kernel via email utilizing tools in git.

After setting up the basic git configs and installing the necessary packages, you are told to configure an **[App Password](https://support.google.com/mail/answer/185833)**, which requires 2FA to function.

As ```@usp.br``` emails do not allow for two factor authentication, the method of setting up the SMTP server described in this tutorial is not valid, so we will skip to the next one.

## Sending patches with git and a usp email

Since 2FA isn't supported for ```@usp.br``` emails, we need to use OAuth 2.0, an authentication protocol.

We are then presented with two methods to configure your git email setup:

- **Method 1 - Email Proxy**
- **Method 2 - ```git credential```**

When I was first doing this tutorial only the former method was available, so I will start by reporting on that one - though I later switched to using ```git credential```.

### Method 1 - Email Proxy

This method utilizes a docker container to act as an email proxy for sending the emails.

The setup requires cloning a [Github project](https://github.com/davidbtadokoro/emailproxy-container) by [David Tadokoro](https://github.com/davidbtadokoro/) that contains the Dockerfile and the ```docker-compose.yaml``` to get the container going.

After configuring the ```emailproxy.config``` file and adding the OAuth tokens, all that's left is setting up ```kw send-patch```.

Then, all we have to do is make sure the container is running before we send the patch.

Overall, as this method requires a container running and quite a lot of configuring, it is less preferred as the next method described below:

### Method 2 - ```git credential```

After the class had already gone through the tutorial utilizing the method above, we were notified about the existence of another way to get ```@usp.br``` emails working with ```git send-mail``` - ```git credential```.

For the setup we make use of a [helper](https://github.com/AdityaGarg8/git-credential-email) that simplifies the process even further.

After setting the OAuth client, we generate the access token. ```git credential``` then stores that token internally, making it so you only need to go through this process once.

Then, all that's left is to configure git to use this credential through OAuth with ```git config --edit```, and we're done.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> It's worth mentioning that, although the tutorial instructs you to configure it locally, I opted to configure it globally so I'd be able to use the email for all of my future projects in this class.
{: .prompt-info}
<!-- markdownlint-restore -->

## Conclusion

The second method streamlines the process and avoids the annoying parts of needing to have a container running in the background to send an email, so setting it up this way felt much easier and cleaner throughout.
