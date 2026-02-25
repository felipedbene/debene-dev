---
title: "From Concept to Execution: My DevOps Journey with Harvester and Salt"
date: 2025-01-15T00:00:00-06:00
draft: false
author: "Felipe De Bene"
description: "Opinions expressed here are my own, fueled by late-night coding sessions and an unreasonable amount of coffee. Building a reproducible DevOps infrastructure with Harvester HCI and SaltStack automation."
tags: ["DevOps", "Harvester", "SaltStack", "Terraform", "Kubernetes", "infrastructure", "automation", "virtualization"]
categories: ["DevOps", "Infrastructure"]
ShowToc: true
TocOpen: false
---

*Disclaimer: Opinions expressed here are my own, fueled by late-night coding sessions and an unreasonable amount of coffee.*

DevOps is all about turning chaos into order and making complex systems look like child's play. My journey started with the lofty goal of becoming a Kubernetes Certified Administrator (CKA), but what started as a study exercise turned into an adventure with two amazing tools: Harvester and Salt. This post is part tutorial, part story, and entirely fueled by enthusiasm. Let's dive in!

## Why Harvester and Salt?

Sure, cloud platforms like AWS are great, but where's the fun if everything's done for you? I wanted something hands-on, something that would push me to the edge of my understanding. That's when Harvester and Salt entered my life. Together, they gave me the flexibility and control I needed to build a rock-solid DevOps setup.

### What I Set Out to Do:

- Build a lightweight, reproducible Linux environment.
- Dynamically configure Kubernetes Control Plane and Worker Nodes at will.
- Automate all the boring stuff.
- Scale infrastructure without spending a dime on cloud services, but thousands on hardware instead.

## Meet the Stars of the Show

### Harvester: Virtualization Made (Almost) Fun

Harvester is a Kubernetes-based hyper-converged infrastructure (HCI) platform. Think of it as the cool, nerdy kid who's great at everything: virtual machines, Terraform integration, and even GPU passthrough (that explains the thousands). It made spinning up VMs feel less like a chore and more like a power trip.

### Salt: Automation's Swiss Army Knife

Salt is the tool I didn't know I needed. It's like AWS SSM on steroids. From bootstrapping nodes to deploying complex configurations, Salt turned repetitive tasks into moments of pure joy. (Yes, automation is joyful. Fight me.)

## How I Pulled It All Together

### 1. Setting Up Terraform

Terraform is the backbone of this setup. To keep things tidy, I ran it inside a Docker container. The configs in the repo let you:

- Define and provision VMs in Harvester with ease.
- Manage node counts and configurations dynamically. Just change a number, and Terraform does the rest.

### 2. Deploying Harvester

Deploying Harvester was refreshingly straightforward. Once it was up, I created tailored machine images with cloud-init. This meant every VM was born ready to work — no awkward teenage phase required.

### 3. Automating Nodes with Salt

Here's where things got spicy:

**Bootstrapping:** With a simple cloud-init script, each node automatically registered with the Salt master. Here's the magic:

```yaml
runcmd:
  - echo "10.0.2.170 salt" >> /etc/hosts
  - curl -o /tmp/bootstrap-salt.sh -L https://github.com/saltstack/salt-bootstrap/releases/latest/download/bootstrap-salt.sh
  - sudo sh /tmp/bootstrap-salt.sh -P stable 3006.1
```

**Node Management:** Salt states handled everything from installing Kubernetes tools to deploying containerized environments. It was like having a superpower.

**Dynamic Scalability:** Need another node? Just bump the Terraform count, and Salt takes care of the rest. It's almost too easy.

### 4. The Salt Master

Even the Salt master wasn't spared from automation. Terraform spun it up, and the `install_master.sh` script in the repo turned it into the command center of my infrastructure. It's like the conductor of an orchestra, except the orchestra is made of servers.

## Cool Things You Can Try

### Adding Nodes

Crank up the node count in Terraform, hit `terraform apply`, and watch as a shiny new VM reports to the Salt master like a good little minion.

### Applying States

Want to turn a node into a Kubernetes powerhouse? Salt's got you covered:

```bash
sudo salt 'minion-9' state.apply setup_kubetools
```

Or maybe you're all about containers:

```bash
sudo salt 'minion-9' state.apply setup_container
```

### Running Commands

Sometimes, you just need to tell all your nodes to jump. With Salt, you can:

```bash
sudo salt '*' cmd.run 'echo "How high?"'
```

Think of the possibilities!

## Why This Matters

This isn't just about learning tools — it's about understanding how to wield them. By combining Harvester and Salt, I built an infrastructure setup that's scalable, reproducible, and just plain cool. Whether you're a DevOps newbie or a seasoned pro, this project has something for everyone.

Ready to geek out? The repo is packed with all the configs and scripts you'll need to get started. Go ahead, take it for a spin. And remember: automation isn't just a skill — it's a lifestyle.

**Explore the Repo:** [https://github.com/felipedbene/tf-harv](https://github.com/felipedbene/tf-harv)

---

*Originally published on [Medium](https://medium.com/@felipedebene/from-concept-to-execution-my-devops-journey-with-harvester-and-salt-85e876353f10).*
