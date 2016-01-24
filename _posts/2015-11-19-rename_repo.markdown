---
layout: post
title:  "Rename the GitHub repository"
date:   2015-11-19
permalink: /rename_repo
---

Your initial repository name may no longer be applicable or you simply desire to change it. This is a very easy process to do.

**1. Navigate to the Settings section of your repository on GitHub and rename your repo.**

**2. Update the remote location `origin` to the new repository name by running:**<br>
`$ git remote set-url origin https://github.com/user-name/new-repo-name.git`

**3. Verify that the change has been made by running:**
`$ git remote -v`

*Bonus! For Heroku:*

Renaming an app on Heroku is as easy as running a single command from the CLI. From within your project directory, run:
`$ heroku apps:rename newappname`