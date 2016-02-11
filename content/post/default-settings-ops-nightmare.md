+++
date = "2016-02-11T19:06:34+05:30"
draft = false
title = "Default Settings: An Ops Nightmare"
banner = "/img/posts/redisallsafe.png"
slug = "default-settings-ops-nightmare"

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["devops", "security", "nvision"]
+++


How the default redis configuration made me lose access to my server.

## Prologue

As a web-coordinator for nvision.org.in, I was making a number of portals for several events. And a general pattern, some of these portals was that either the portal needed updates while the event was happening or we needed to change some data in a flat-file database. These portals were made in express with an in-memory cookie store.

The problem was that whenever I pushed an update, people were logged out. This was especially a problem if I was pushing constant updates trying to fix a bug in production as it occurred in another event Celestial Inquisition.

For our flagship online event Eldorado (http://eldorado.nvision.org.in), I wanted to push updates without logging people out. This involved using a persistent cookie store and I chose redis for that.

## What happened?

  I installed redis using apt-get and generally assumed that the default configuration was good enough. I was getting login persistence across restarts and the event went really well....

  Only to realize that I couldn't login the next morning. After trying several methods, I still couldn't get my server to accept my ssh key. I had to shut it down from the Digital Ocean dashboard and manually reset the password.

  After logging in, I realized that my `~/.ssh/authorized_keys` file was filled with garbage. A quick google search told me that redis could have caused that.

  By default redis listens on `0.0.0.0` and it has no authentication. And some pirate-crawler-bot on the internet discovered it and filled my authorizedkeys with garbage. I then restarted redis with the right configuration and refilled my authorized_keys fixing the issue once and for all.

## Epilogue

Redis is an awesome software. I don't blame redis, but rather myself for not reading the deploy to production document (http://redis.io/topics/admin).

While the post above looks like its no big deal (I have a lot of Ops pride), I was freaking out when I lost access. DigitalOcean saved my arse (@DigitalOcean <3). If I couldn't gain access, we would have withdrawn Eldorado, a game played and enjoyed by tons of people.

I learnt one huge lesson out of this one:

    ALWAYS check the configuration file. If possible rewrite yourself it to be safe.

## One More Thing

*Idea: Reverse Crawler*

One thing became evident to me. There are a lot of bots on the internet trying to exploit weak servers. I once got a notice from DigitalOcean about a droplet that was being used to send DoS attack. Logs showed that it was because of a weak password which was brute-forced by a server in China. I completely shunned passwords and moved to key-based auth from then.

Now looks like the bots also look for common vulnerabilities.

How about we write a reverse proxy that listens on all ports and adds the list of bots to a black-list. This way we can come to an estimation of the number of bots out there and also which ports and vulnerabilities they generally look for.

I see some glaring implementation problems but I think it can be done if the internal services listen on an internal software interface.