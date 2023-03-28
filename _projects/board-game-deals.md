---
title: "Board Game Deals discord server"
order: 1
header:
  teaser: /assets/images/projects/board-game-deals-teaser.png
sidebar:
  - title: "Tech stack"
    text: "Node.js, Google Cloud"
  - title: "Status"
    text: "Active (2022 - Present)"
  - title: "Demo"
    text: "[Join the server](https://discord.gg/dgNNechKrQ)"
gallery:
  - url: /assets/images/projects/board-game-deals-gallery-1.png
    image_path: /assets/images/projects/board-game-deals-gallery-1.png
  - url: /assets/images/projects/board-game-deals-gallery-2.png
    image_path: /assets/images/projects/board-game-deals-gallery-2.png
---

I'm the creator of a discord server for board game lovers 🎲 ([Twilight Imperium](https://boardgamegeek.com/boardgame/233078/twilight-imperium-fourth-edition), one love!).

{% include gallery %}

The technical side of it consists of a backend app (Node.js) that is running in Google Cloud and posts daily updates on sales, auctions, drops, and upcoming crowdfunding.

The Node.js app acts as a REST API (App Engine), that is called regularly with Cloud Scheduler jobs (frequency is set using CRON).

![image-left](/assets/images/projects/board-game-deals-google-cloud.png)

Whenever changes are pushed to the `main` branch, the app is built (includes compilation, testing, and static analysis) and deployed to production via a CI/CD pipeline set up using GitHub actions and Google Cloud Build (cloudbuild.yaml is used, [read more](https://cloud.google.com/build/docs/configuring-builds/create-basic-configuration)).

![image-left](/assets/images/projects/board-game-deals-google-scheduler.png)

To save time and resources on parsing and monitoring Amazon deals and board game deals, keepa.com service is utilized. By adding the desired items to track, it sends notifications for any price drops.

![image-left](/assets/images/projects/board-game-deals-keepa.png)

In addition to Discord, the integration also includes posting to the [tabletopgamedeals](https://www.reddit.com/r/tabletopgamedeals/) subreddit on Reddit. This ensures that even those who are not familiar with Discord can subscribe to get notified about the latest deals. The [reddit npm package](https://www.npmjs.com/package/reddit) has been used for this purpose.

![image-left](/assets/images/projects/board-game-deals-reddit.png)

## Skills

- Node.js
- Google Cloud
- JavaScript
- Solution Architecture
- Backend Development
- API Development
- Git
- CI/CD
