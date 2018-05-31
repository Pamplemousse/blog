---
layout: post
title: Google Hangouts with Irssi on Nixos
tags: [ nixos, irssi ]
---

As I wanted to have access to Google Hangouts chats with [Irssi](https://irssi.org/) on [NixOS](https://nixos.org/), here is a write-up of how I got it working.


## The protagonists

After a quick research using my favorite search engine [DuckDuckGo](https://lmddgtfy.net/?q=irssi%20google%20hangouts), it turns out that we will need to add two piece of software to Irssi.

  * [BitlBee](https://www.bitlbee.org/main.php/news.r.html): an IRC gateway that act as a server your client connects to, using the IRC protocol ; and "translates" what you send and receive to another protocol (depending on whom your gateway connects to)
  * [purple-hangouts](https://bitbucket.org/EionRobb/purple-hangouts): a library "to support the proprietary protocol that Google uses for its Hangouts service"


## Add them to the system

We are pretty lucky as packages for [BitlBee](https://nixos.org/nixos/packages.html#bitlbee) and [purple-hangout](https://nixos.org/nixos/packages.html#purple-hangout) are available on NixOS.

However, `purple` is not a plugin installed by default in the `bitlbee` package: we need to declare that we want it enabled.

Having a look at the [declaration of `bitlbee`](https://github.com/NixOS/nixpkgs/blob/8aa385069f830fc801c8a04d2bd8a70a02be3de4/pkgs/applications/networking/instant-messengers/bitlbee/default.nix#L27), we can find out the name of the relevant build option.

Let's edit `/etc/nixos/configuration.nix`:
```nix
environment.systemPackages = with pkgs; [
  [...]
  bitlbee
  purple-hangouts
  [...]
];

nixpkgs.config.bitlbee.enableLibPurple = true;

services.bitlbee = {
  enable = true;
  libpurple_plugins = [ pkgs.purple-hangout ];
};
```

**You can see how this fit my whole configuration [on my GitHub repo](https://github.com/Pamplemousse/laptop/blob/master/etc/nixos/configuration.nix)**.

Then rebuild the system:
```bash
sudo nixos-rebuild switch
```


## Try it out

See how it goes: start `irssi`, then type the following commands:
```irc
<@pamplemousse> /connect localhost
<@pamplemousse> /join &bitlbee
```

At this point, I need to create an account to identify myself to the BitlBee server.
```irc
<@pamplemousse> register StrongPasswordGeneratedWithKeepassXC
```

And verify our the plugin to communicate with Google Hangouts is present:
```irc
<@pamplemousse> plugins
[...]
<@root> Enabled Protocols: aim, bonjour, gg, hangouts, icq, identica, irc, jabber, novell, oscar, simple, twitter, zephysr
```

All good!


## Setting up BitlBee to access your Google Hangouts account

```irc
<@pamplemousse> account add hangouts MyAddress@Email.Com
<@pamplemousse> acc hangouts on
```

The next step is one of the most unreliable thing I have ever done to configure an account.

In fact, the previous command created another Irssi window to interact with the lib (that is, a private conversation with `purple_request_0`).

Follow the instruction that appeared there, and **reply the oauth code that you obtain in the conversation** (took me 30 minutes to figure this out).

Once that's done, you should see all your contacts appearing in the `&bitlbee` window.

## Try it out

We can now start 1-on-1 conversations, for example with JohnDoe:
```irc
<@pamplemousse> /msg JohnDoe hello
```

And even join group chats (that exists):
```irc
<@pamplemousse> help chat list
<@pamplemousse> chat list hangouts
<@pamplemousse> chat add hangouts !1 #chatname
```

So later on, we can use the shortcut `#chatname`:
```irc
<@pamplemousse> /j #chatname
```

**That's it! We now can chat on Google Hangouts from Irssi!**
