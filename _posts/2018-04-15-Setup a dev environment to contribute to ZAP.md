---
layout: post
title: Setup a dev environment to contribute to ZAP
---

## Foreword

[ZAP, or Zed Attack Proxy](https://www.zaproxy.org/), is an [OWASP](https://www.owasp.org) project to make a free security tool to help developers and security experts test and find vulnerabilities in web applications.

I have been given the opportunity to contribute to it, and, being an open-source project, I feel like it would be a good idea to share my tribulations.
I ambitiously hope it will reduce the time and effort potential future contributors would have to invest diving into it.

The subject is so wide I will not be able to cover it entirely in this single blog post.
There is enough content to start a series, and I should stay motivated to write about it: stay tuned.

For now, let's start with the basics.

**How to get the code running on my machine?**


## First steps

Documentation about ZAP development can be found on the [Zap's repository wiki](https://github.com/zaproxy/zaproxy/wiki/Development). Anything I will present here have been found roaming around the docs.

Let's start by cloning the main repository, build ZAP and start it from the command line.

```bash
mkdir zap && cd zap
git clone git@github.com:zaproxy/zaproxy.git

ant -f zaproxy/build/build.xml dist
./zaproxy/build/zap/zap.sh
```

And then we have ZAP running. Smooth.


## Extensions, Add-ons

Lots of ZAP's "logic" has been extracted from the core repo into so-called add-ons, which are located into the [`zap-extensions` repo](https://github.com/zaproxy/zap-extensions).

Here you can find a general overview about Add-ons: [github.com/zaproxy/zap-core-help/wiki/HelpStartConceptsAddons](https://github.com/zaproxy/zap-core-help/wiki/HelpStartConceptsAddons).

As we are going to work on these as well, let's clone the repository alongside the `zaproxy` one.

```
git clone git@github.com:zaproxy/zap-extensions.git
```

Again, [its related wiki](https://github.com/zaproxy/zap-extensions/wiki) might be of good help.

There are (as far as I know), two ways to get add-ons in ZAP: via the "marketplace" (located in "Manage Add-ons" in the "Top Level Toolbar") or load them from a file.

In our case, as we want to edit add-ons and watch their brand new behaviour, we will use the latter.

In an upcoming post, we will talk about how to bring changes to an Add-on, but before that, let's ensure we can build them normally.


### Build and use the Add-on "as is"

As we are going to have a look at a specific issue related to [Zest](https://github.com/mozilla/zest), this is the add-on we are going to look at.

Depending on the extension we want to work on, we gonna have to checkout the related branch. In our case, we want to work on Zest, so let's checkout the `beta` branch, and build this Add-on specifically.

```
cd zap-extensions
git checkout beta
ant -f build/build.xml deploy-zest
```

Then, in ZAP, press `Ctrl+l` (or go to "File > Load Add-on File"), then select the brand new plugin file (usually, it has been deployed to the "zap/zaproxy/src/plugin/" folder).

**At this point, you should have the Zest extension working fine in ZAP, and that's enough for today.**
