---
title: "A first month: Scaling a 25.000 user Mastodon instance using Kubernetes"
date: 2022-12-03T09:00:00
draft: false
url: /a-first-month-scaling-a-25-000-user-mastodon-instance-using-kubernetes/
tags:
  - mastodon
---

{{< imgh src="mastodon-scaling-post-header.jpeg" alt="Two phones with the Mastodon app open" >}}

Today marks the first entire month of my being a Mastodon server administrator.
With Elon's recent purchase, a lot of people went on a search for alternatives,
and they found one: [Mastodon](https://joinmastodon.org).

<!--more-->

The critical difference with Mastodon is that everyone can own a social media
platform, and at the same time, no one can. It's a decentralized platform,
communicating with other instances in the fediverse.

I love everything technical and jumped at the opportunity to start my instance.
The days since were a wild ride. If memory serves me well, the first days went
like this:

- November 4th: 60 users
- November 5th: 600 users
- November 6th: 6000 users
- November 7th: 15,000 users

You get the essence. It grew like crazy. Luckily, with me working primarily with
Kubernetes, I decided from the beginning to set up Mastodon's infrastructure
using it. Still, I spent most of the first week getting bigger database servers
and node pools. That's also when I called for help from my good friend Mijndert
Stuij. Combining our background, we have over 25 years of experience managing
online software at scale.

Early on, we wanted to open-source everything we did for my now-grown instance,
[toot.community](https://toot.community). [Mijndert](https://mijndertstuij.nl)
adopted all the infrastructure into Terraform, and I cleaned up the Kubernetes
manifests so it was shareable with the world.

It's noteworthy that after an entire month of running Mastodon, our
infrastructure and configuration are finally ready for the world to see:

- **GitHub**: https://github.com/toot-community
- **Kubernetes**: https://github.com/toot-community/kubernetes
- **Platform**: https://github.com/toot-community/platform
- **Blogpost**: https://blog.toot.community/posts/open-sourcing-toot-community/
