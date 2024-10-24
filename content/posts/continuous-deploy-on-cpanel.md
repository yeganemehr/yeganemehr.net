---
title: "Continuous Deployment on Shared Hosting"
date: 2024-10-24T17:20:00+01:00
---

Like a lot of modern web developers, I rely heavily on containers to get things up and running in production.
I’ve gotten so used to it that I pretty much forgot about the good-old days of shared hosting.

But today, I worked on a project that was still running on a cPanel host, with no access to shell, SFTP, or SCP.   
The site was built with [WHMCS](https://www.whmcs.com/), and the client was making changes directly to the production code every day!!

{{<image src="https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExbmVkN2pqOW81ZDVhZ2w2M3ZhOTVoYzBucTltbTEzOGc4dW00b2ZscyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/xUA7bcUzBNSLC8Zy3C/giphy.webp" alt="Disgusted">}}


## Problem

Before we get into the details, let’s talk about what’s wrong with this setup and why deployment automation is a must.

I'll use this project as an example, but the issues are common across many other setups.

So, Mr. Wilson (the client) has been managing his WHMCS-based site for years. He’s installed all sorts of modules and uses a custom template that gets updated pretty often.   
Now that his business is growing, he brought my team in to add some new features.

We set up a GitHub repository to manage the code and worked on multiple issues and branches simultaneously. However, each deployment required us to manually follow these steps:

1. Download the repository’s zip file.
2. Remove non-production files.
3. Upload it using cPanel’s File Manager.
4. Extract the zip to overwrite the current version.
5. Manually check for deleted files in recent commits and remove them one by one.

{{<image src="https://media.giphy.com/media/j0gQA2VD38NKc9rc8y/giphy.gif" alt="Bye-Bye">}}



Any experienced engineer would agree that this process is unacceptable. It was time-consuming and risky, there were plenty of opportunities for things to go wrong.

While using GitHub as our version control system (VCS) and issue tracker improved the workflow, we created a new problem by manually deploying every commit to the production server.


## Solution

We needed a solution to automate those five steps.
Without shell access to the host, I saw two options:

A: **Write a script** to download the GitHub repository’s zip file and perform the update steps when triggered by a GitHub webhook.

While straightforward, this approach had several downsides:
1. The script would be another component to develop and maintain (cost factor).
2. It would need to be publicly accessible over the internet for GitHub to reach it (security risk).
3. Setting up and maintaining this updater would add complexity.

So, I ruled this out.


B: **Use GitHub Actions** to upload the repository’s content directly to the host.

I preferred this option since it required no dependency on the production environment and gave me flexibility for future preprocessing or bundling template assets.

{{<image src="https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExbDQ5d2cyNjVlOXU2NHZiMnhzMXV6NzJ2YXBjb3VmcTlwNDVrYTd4aSZlcD12MV9naWZzX3NlYXJjaCZjdD1n/tIeCLkB8geYtW/giphy.gif" alt="Thums-up">}}

With a clear plan, setting up continuous deployment wasn’t too difficult.

I began with the following workflow, with comments to explain key parts:

```yaml
name: Deployment


on:
  push:
    # Run this workflow when a tag in the format "v*.*.*" (like v1.0.2) is pushed
    tags: 
      - "v*.*.*"

    # Allow manual workflow triggers
    workflow_dispatch:

jobs:
  deploy:
    name: Deploy Production
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

# Remove files we don’t want to push to production, like docs or, in this case, email templates
      - name: Remove extra files
        run: rm -fr .github templates/emails 

# Upload the current directory to the public_html directory on the host
      - name: Upload ftp
        uses: genietim/ftp-action@releases/v4
        with:
          host: hostname.com
          user: the-username
          password: ${{ secrets.FTP_PASSWORD }} # Stored in GitHub Secrets
          remoteDir: public_html
```

However, after running this a few times, I discovered that the [genietim/ftp-action](https://github.com/marketplace/actions/ftp-upload-action) was stateless—it didn’t remove deleted files from the repository.

This was a major issue, so I searched for an alternative and found [SamKirkland/FTP-Deploy-Action](https://github.com/marketplace/actions/ftp-deploy), which provided the extra features I needed: local file exclusion and state tracking for remote files.


Here’s my updated workflow:

```yaml
name: Deployment
on:
  push:
    tags:
      - "v*.*.*"
    workflow_dispatch:
jobs:
  deploy:
    name: Deploy Production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Upload ftp
        uses: SamKirkland/FTP-Deploy-Action@v4.3.5
        with:
          server: hostname.com
          username: the-username
          password: ${{ secrets.FTP_PASSWORD }}
          server-dir: public_html
          state-name: .ftp-deploy-sync-state.json # Tracks which files have been deployed
          exclude: |
            .git*/**
            templates/emails/**
```

Now, with every new tag or release, the project is deployed automatically in seconds. There are no extra costs, no human errors, and it all runs on an affordable shared hosting account.

## Final Thoughts

If you're still doing manual deployments, you're not only wasting valuable time but also increasing the likelihood of errors.   
Automating your deployments with GitHub Actions is a simple yet powerful way to streamline your workflow, minimize risks, and avoid the pitfalls of manual updates.

Even if you're on a shared hosting plan, automation is well within reach and can make a world of difference in how you manage your projects.   
The time and effort invested in setting up continuous deployment will pay off in the long run, saving you from countless headaches and ensuring your deployments are fast, reliable, and error-free.

{{< notice tip >}}
It’s an investment worth making for any project, no matter the scale.
{{< /notice >}}

