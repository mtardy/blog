---
title: "Kubernetes and HostPath, a Love-Hate Relationship"
date: "2022-03-03"
description: ""
tags: ["kubernetes", "security", "cve"]
categories: ["kubernetes"]
url: "posts/k8s-hostpath"
canonicalURL: "https://blog.quarkslab.com/kubernetes-and-hostpath-a-love-hate-relationship.html"
ShowCanonicalLink: true
---

The article is available on [Quarkslab's
blog](https://blog.quarkslab.com/kubernetes-and-hostpath-a-love-hate-relationship.html).

It traces the history of three Kubernetes-related vulnerabilities. Explaining
what they are, how they were patched, and how they are related. The
exploitation of these vulnerabilities allowed access to the underlying host
filesystem for users that were not properly authorized.
