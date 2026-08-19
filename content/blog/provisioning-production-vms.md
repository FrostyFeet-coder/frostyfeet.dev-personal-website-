---
title: "Lessons Learned: Provisioning Production VMs for Node & Python"
date: "2026-08-19"
description: "A look back at the hurdles of provisioning cloud VMs, avoiding OOM crashes, and managing Python environments alongside Node.js."
---

Provisioning virtual machines for production isn't as simple as `apt-get install nodejs` and walking away. Yesterday, I spent time configuring and refining the setup process for a new production VM on Google Cloud (specifically, an `africa-south1-b` instance for the `lelapa-ai` environment). 

Here are the key takeaways and hard-learned lessons from setting up a server that runs both a Node.js backend and multiple Python machine learning pipelines.

## 1. Pin Everything (And I Mean Everything)

When writing bootstrap scripts (`bootstrap-vm-packages.sh` or `setup-conda-envs.sh`), relying on `latest` is a ticking time bomb. I made sure to pin the exact runtime versions:
- **Node.js**: 22.17.0 (with npm 10.9.2)
- **PM2**: 7.0.1
- **Conda**: 26.3.2 (Miniconda3-py313)

If you don't pin these, a deploy script that worked flawlessly in staging six months ago will fail silently in production because a package manager updated its default repository.

## 2. Managing Multiple Python Environments

Our backend relies on different ML tools that require wildly different Python versions (e.g., an `autotransform` pipeline requiring Python 3.11.4 and a `silero` audio model requiring Python 3.8.10). 

Using the system Python is a recipe for disaster. The best solution I've found is installing Miniconda locally to the SSH user (e.g., `/home/karya/miniconda3`). 
- Leave the `base` environment completely alone (this acts as your login shell fallback).
- Create dedicated conda environments for each pipeline.
- When your Node application needs to invoke a Python script, it should explicitly call the binary from the environment: `~/miniconda3/envs/autotransform/bin/python script.py`.

## 3. The Monorepo Build Trap (OOM Crashes)

This was perhaps the most painful lesson. If you have a large monorepo (e.g., managed by Lerna or Turborepo) and you run a full build command like `npm run build` or `lerna run build` directly on the VM, you are highly likely to encounter an **Out of Memory (OOM) crash**.

Full builds run cleanup scripts, spin up multiple parallel TypeScript compilers, and consume massive amounts of RAM. Small-to-medium cloud VMs will simply panic and kill the process.

**The Fix:** 
If you absolutely must compile on the VM, use scoped compilation. Never build the whole repo. For instance:
`npx lerna run compile --scope=@myorg/backend`

(Even better: Build your code in a CI/CD pipeline like GitHub Actions and only ship the compiled artifacts to the VM, though that requires more infrastructure setup).

## 4. Graduating from Tmux to PM2

In early development stages, it's common to SSH into a dev server and run services inside `tmux` sessions. While great for debugging, it's terrible for production. If the VM reboots or a service panics, it stays dead. 

Migrating to a `pm2 ecosystem.config.js` file ensures that your Node backend, frontend server (like a locally served Next.js/React app proxied through NGINX), and any background workers are automatically restarted on failure and boot up on system startup.

---

Automating infrastructure is hard, but documenting these constraints—especially the quirky ones like scoped compilation to avoid OOMs—makes spinning up the next environment much less painful.
