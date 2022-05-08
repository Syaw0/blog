---
layout: post
title: "How to handle version mismatch between client and server?"
date: 2022-04-22 18:21:10 +0900
categories: Front-End
image: /static/version-mismatch.png
---

## The Problem

Yeah, I met this problem recently about how to handle version mismatch between client and server.

Say we are serving version 1 on both client and server.

![](/static/version-mismatch-1.png)

Everything works fine, since all version are matched.

Now we are going to bump to version 2. But we are facing a problem: **There could be a version mismatch between client and server**, like the illustration below.

![](/static/version-mismatch-3.png)

It could be led by following reasons:

#### 1. version mixed among servers

When we have multiple servers exposed through load balancer, since we cannot deploy new version to all server at same time, there could be a (relatively short) period of time that two version both exist among our servers, when requests from clients land, it might land to different versions.

This could result in the mismatch of `Old client <-> New Server` and `New client <-> Old Server`, which is colored in below illustration

#### 2. stale static resources on client

Yeah, static resources are often cached, by CDN/browser/Service Workers, if users don't refresh the web page, it is not refetched and stays at the old version.

This could result in the mismatch of `Old client <-> New Server`

## Impact of the Problem

The impact of this problem varies on your app's architecture, if you only have one server and not much client-side work, then I guess it is fine.

Yet for most of us building rich web apps on the client-side, generally this is something we should avoid.

I categorize the impact into 2 categories based on what your server is serving.

1. **pure data dependencies**: Meaning your server is only serving data through API, and the initial bootstrap code. This is clean separation.

2. **component dependencies**:This is more complex, for example if you are dynamically serving page route because of some authentication .etc. Generally it means client-side loader send a request and serve returns some script.

From the client-side perspective, any unexpected change would result in a unexpected state. It could results in errors, or users unable to finish some interaction flows.

**versions between client and server should be matched, at least mixing state should be avoided**.

## How to handle the problem?

### Sticky Sessions on Load Balancers

[Sticky Sessions](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html) on load balancers could ensure client-side requests always land on the same server.

This solves the `New Client <-> Old Server` case, so we only need to focus on `Old Client <-> New Server` case

![](/static/version-mismatch-4.png)

### Let server know current client version

This is a concept pretty familiar in the API versioning.

When we we change the API response

1. if to add some fields for new client => we can safely add them
2. if to modify or remove some fields => version it.

If there is only data dependencies being serve, then API versioning is enough.

For component dependencies, we can achive something similar.

1. when build our static resources, store current the version in the server
2. when serving some dynamic stuff, compare the client-side version from cookie and the version needed
3. serve differently based on the results (or fallback plan, you can force showing a popup to ask users to refresh)
4. remove the backward compatibility after some time.

For above to work, we need to make sure old static resources are served as well on our servers.

Just like we cannot support all old versions of Native Apps, we cannot keep the backward compatibility for ever. So we need to delete them after some time, this could be done by monitering the version passed along, by simply counting them and set up a threshold.

## Does hash-based versioning help?

By has-based versioning I mean we hash each module based on its content, which results different versions for each module rather than a single version number.

It could save us some storage because we are only building the modules that is modified, but storage is cheap, not that helpful.

Speaking of backward compatibility, it actually doesn't help much, because we are still facing the issue, but rather than one single versioning, we are comparing on a version graph.

It doesn't help because we still need to differentiate old client and serve the old version component anyway.

## How does Next.js solve it?

Not sure, I guess not and I'd love to find out but first I need to understand how Next.js bundles its resources.

# Summary

The problem I mentioned really depends on how you architect your Apps, it might not be a problem, but the general idea should hold true, that is we need to avoid mixed version in the whole user experience.

The way to solve it is to **let server know what version current client is using and do backward compatibility**
