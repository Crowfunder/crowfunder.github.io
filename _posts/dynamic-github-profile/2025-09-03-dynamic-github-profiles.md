---
title: Making GitHub Profiles Cool - Painful Lessons with GitHub
date: 2025-04-06T15:53:47.000Z
modified: 2025-09-04T19:01:47.000Z
tags:
  - github
  - github-actions
description: >-
  Discovering a simple way to have a dynamic, free, customized GitHub profile
  reacting to, for an instance, time of the day.
---
As I was preparing for yet another job search, I've decided it's about time I write a proper, neat README for my GitHub profile. It's common knowledge how professionalism is measured by the number of stars on your repos, how green your activity graph is and how cool your profile looks.

The "coolness" of the profile, while entirely subjective, can be derived from numerous things. For some people, it's denounced by how many animated stat graphs you can fit on one page, for others it's how niche the anime girl on your avatar is, but I thought it would be neat to have a "technically impressive" (/s) profile.

## The Idea

Recruiters, just like programmers or other human beings, tend to work at various times of the day. I thought it would be a cool idea to, bear with me, "personalize" the experience a bit. It's a common thing with (human) languages how you greet people in different ways depending on the time of the day. 

Let's say I want to have a "different" profile in the day and in the night - for starters, different image banners. Well, while GitHub markdown does support HTML tags, it doesn't support any code execution. The simple way would be a small server that serves different resources based on the time of day, but guess it or not, I'm a broke student and I'd rather not spend money just to have a day/night banner.

**TL;DR Use GitHub Actions to modify user profile repository by substituting linked assets, caching shenanigans ensue.**

**EDIT: It's been a while since I wrote this article and to my minor dismay, it turns out that it is not an entirely new idea and is quite well documented. Check out [this repository](https://github.com/abhisheknaiidu/awesome-github-profile-readme?tab=readme-ov-file#github-actions-) for more, cool examples on GitHub Actions integrations with your profile.**

## GitHub Actions to Action
If I know a way to have a free (ish) "server" that performs tasks, to that one that has a capability of interacting with GitHub repositories, it's GitHub Actions.

Love them or not, they have a [very generous free tier](https://github.com/pricing) - 2000 CI minutes per month. For scale, the runners used by this project take up 2\*3=6 seconds per day, which results in around 3 minutes of usage per month, out of 2000. Neat, isn't it? That is for public repositories though, but for our use case it's not a problem, as the "CDN" needs to be public anyways.

<figure>
<img src="{{site.imageurl}}/_posts/dynamic-github-profile/free-tier.png" alt="github free tier perks" style="max-width: 55vw;">
<figcaption>Github free perks, including free CI minutes.</figcaption>
</figure>


Connect that with the fact that GitHub repositories are essentially free glorified file storage, and we have the stack for our little project ready. 

*Note: I am, of course, NOT affiliated with GitHub in any way, Even less so, as mentioned above I'm **looking** for a goddamn job.*

## Implementation
Let's start by creating a simple animated banner that changes with the time of the day. 

The repository that serves as our CDN (coincidentally, also the profile README repo, but it isn't a necessity) has an "assets" subdir, with a nested "parts" subdir. The "assets" subdir will hold a "banner.gif" file that will be linked in the profile and will be overwritten by GitHub Action with files from "parts" subdir. 

<figure>
<img src="{{site.imageurl}}/_posts/dynamic-github-profile/files.png" alt="files structure" style="max-width: 55vw;">
<figcaption>Repository file tree</figcaption>
</figure>


Once we have the files ready, let's figure out how to get an action to run on schedule. A quick Google search (and a dig through AI generated slop) led me to [this amazing blog post](https://jasonet.co/posts/scheduled-actions/) by [Jason Etcovitch](https://twitter.com/JasonEtco) which gave a detailed answer. It did not however mention the caveats, but we'll get to that later.

<figure>
<img src="{{site.imageurl}}/_posts/dynamic-github-profile/cron-syntax.png" alt="cron job syntax" style="max-width: 55vw;">
<figcaption>Cron syntax for scheduling actions</figcaption>
</figure>

As it turns out, Actions can be scheduled like a good-old Cron job. Great, we can create two actions - one that runs in the morning, the other that runs in the evening. Actions are also capable of executing shell commands on their runners, and performing any kind of tasks with the repository; for our case - committing and pushing.

## The Code

```yaml
name: Set Day Image

on:
  schedule:
    - cron: '0 6 * * *'  # Runs at 6:00 AM UTC every day
  workflow_dispatch:      # Allows manual run from GitHub UI

jobs:
  update-image:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repo
        uses: actions/checkout@v4

      - name: Replace with Day Image
        run: cp assets/parts/day.gif assets/banner.gif

      - name: Commit Changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add assets/banner.gif
          git commit -m "Update image to day version" || echo "No changes"
          git push
```

The first, vital element is the scheduling. The `on:` parameter defines triggers that run the action (Check out [this](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows) page of actions documentation for other triggers). In our case, it's the "schedule" trigger.  The other trigger, `workflow_dispatch` allows triggering the action manually, with a press of a button. ([docs](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/manually-running-a-workflow))

*PS: Thinking back, if I read the documentation instead of relying on just the previously mentioned blog post, I would know of at least one issue with actions.*

I would also like to avoid spamming my activity graph with regularly scheduled commits, since let's be honest, artificial activity doesn't look very good. I recalled seeing repos that have commits by Actions Bot, quick look into the checkout action documentation results in [this](https://github.com/actions/checkout?tab=readme-ov-file#push-a-commit-using-the-built-in-token) , a GitHub Actions bot git identity. 

The rest is simple, copy the appropriate file, overwriting "banner.gif", commit and push. What could go wrong?

## The "oh no" Moment
The first issue I encountered was fairly apparent, action failed with some permission errors. 
> Permission to git denied to github-actions[bot]

A quick search gives a [quick resolution](https://github.com/ad-m/github-push-action/issues/96), apparently Actions are given read-only permissions to the repo by default. All that needs to be done is giving the workflows "Read and write permissions" in repository settings.

<figure>
<img src="https://user-images.githubusercontent.com/2881159/127678772-776865c9-ca74-449b-a6df-8e95a0471560.png" alt="what to click in actions settings" style="max-width: 55vw;">
<figcaption>What to click to change Actions permissions</figcaption>
</figure>

Reasonable, right? With the read-write permissions the action ran smoothly, replacing the file correctly. 

Now, it's time to see if the scheduling works properly. Evening was nearing by, so I've decided that I can verify personally if the scheduling really works. The action was scheduled to run at 6PM UTC, the clock showed 6:05, the action wasn't running. 

6:15, "what the hell?" I muttered, as I was double checking my Action configs and files, wondering what was it that I messed up again. I concluded there's no use in waiting, I ran the action manually again and it worked as expected. The file was successfully replaced, commit authored by GitHub Actions Bot. I started digging.

Surprisingly, it didn't take long for me to find a [discussion](https://github.com/orgs/community/discussions/147369) referencing a relevant [blog post](https://upptime.js.org/blog/2021/01/22/github-actions-schedule-not-working/). It mentions that scheduled GitHub actions barely ever run on-time, with delays up to several minutes being the norm. As it turns out, the scheduling documentation mentions the same thing, but nobody ever reads the docs, right?

<figure>
<img src="{{site.imageurl}}/_posts/dynamic-github-profile/actions-schedule-discussion.png" alt="discussion on cron jobs syntax" style="max-width: 55vw;">
<figcaption>Reply of a Github affiliate, exerpt from the linked post</figcaption>
</figure>

Honestly, other than being salty for deepening my paranoia, it doesn't matter much. This isn't a time-critical action that needs to run at exact times - it being late by even an hour isn't really a problem. Still, it's worth knowing in case you're doing something that requires the scheduling to be punctual. The article linked above mentions that the only way to assure the actions run on schedule is to employ an external scheduling server that invokes the GitHub API for running actions manually.

I re-scheduled the action to run within an hour for testing and so it did, with a delay of course. Now, it was time to finally see the fruit of my labor.

## Cache, oh a double-edged sword
If you've ever linked to an external image on GitHub, you may have noticed how upon hovering over it its URL shows as something along the lines of `camo.githubuserconent.com` instead of the link you used. That's because, by default, GitHub caches static images and queries cache instead of the actual URL in order to "preserve anonymity". Generally speaking great idea, but for our case it's more of an issue than anything. [Read more](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-anonymized-urls) 

That's exactly the issue I had, but at that time I lacked this information. You can imagine my genuine confusion when upon confirming the banner got replaced by the action, the old image was still there. I saw a different image on my profile and in the repository, where both were supposed to be the same file. To make the matters worse, it didn't matter where I opened the README, it has shown the same thing. 

Well, now we're in quite a pickle. I started digging for why the images in README are not updating. I found this [discussion](https://github.com/orgs/community/discussions/46773) and it remarked using a PURGE http method, which the GitHub API apparently understands (I've not once heard of that method in my entire lifetime). But that solution is a little tedious - manually invoking the API twice a day is, generally speaking, not the best solution. Automating it was also uncertain, as I wasn't sure if the cache URLs were really static, especially after purging cache. 

Fortunately enough, yet another, surprisingly simple [solution](https://github.com/atom/markdown-preview/issues/207#issuecomment-248848108) was proposed - appending a question mark at the end of the image URL (if it isn't already parameterized, if it is then it already shouldn't cache the image) . As stupidly simple as it is, it warrants a cache miss every single time. Upon applying this quick tweak, the image changes were finally visible. The profile was complete... or was it?

<figure>
<img src="{{site.imageurl}}/_posts/dynamic-github-profile/profile-end-result.png" alt="github profile after changes" style="max-width: 55vw;">
<figcaption>The "end" result, looks neat, doesn't it?</figcaption>
</figure>

## What's next?
Knowing that you can freely modify your README.md with scheduled arbitrary code execution, I think you can imagine that anything is possible. For example, another great idea I had in mind was displaying the weather in my city for recruiters to indulge themselves in it (be it out of envy or schadenfreude, mostly the latter though). 

Generous free tiers are amazing, especially for CI/CD which will inevitably be abused, be it for building malware, or for swapping images on a GitHub profile.
