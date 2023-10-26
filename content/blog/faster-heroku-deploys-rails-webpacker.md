---
title: Faster Heroku deploys with Rails and webpacker
date: 2019-08-27
description: Making crazy long Heroku deploys faster
tags: ["heroku", "devops", "rails", "webpacker", "tips"]
---

Some time ago in [Pilot](https://pilot.co/), we switched to use [webpacker](https://github.com/rails/webpacker) as a tool for compiling our JavaScript code. It is a simple way to integrate webpack into your Rails project.

Lately, we noticed that our Heroku deploys got frustratingly long. Over 10 minutes to build and release our main application wasn't acceptable. Add CI runtime and Heroku's [preboot](https://devcenter.heroku.com/articles/preboot#deploying-with-preboot) to the mix and now we could easily end up with around 25-30 minutes of full deploy time. This was getting very annoying when trying to ship a hotfix.

What if we could make it shorter?

## What's going on?

The biggest chunk of time in the build process has been asset compilation. With each build, Heroku was installing all npm dependencies (using yarn) and compiling webpacker assets from scratch. It took very long with each build, even if we didn't touch any code (deploying the same code twice).

Something didn't smell right as on local dev environment if there were no changes, the process ran in less than a second.

As time passed, we also noticed that our slug size was growing, which causes deploys to be longer. There were lots of uncleaned leftovers after each webpacker build. Simple [repo purging](https://github.com/heroku/heroku-repo) was enough to get rid of this problem, but it required manual command execution.

It was clear that there are a couple of problems:

- Yarn cache isn't re-used.
- Full webpacker build runs even if JavaScript code was not touched.
- Old build products were not cleaned.

## Getting to a solution

Luckily some people already [worked on a solution](https://github.com/heroku/heroku-buildpack-ruby/pull/892). With that webpacker will reuse the previous build if JavaScript files were not touched. Otherwise, webpacker will do a full rebuild. On top of that, these changes include a Yarn cache, which remedies problems with dependencies installation (however this could be achieved installing official node.js buildpack from Heroku).

Another problem is that webpacker [does not ship with a cleaning task](https://github.com/rails/webpacker/issues/1410) (similar to `assets:clean``). Without it, the slug size of our application will keep growing in time. Luckily for us, there is a webpack plugin called [clean-webpack-plugin](https://github.com/johnagan/clean-webpack-plugin) that does exactly this.

Combining that plugin and making a little tweak in the [solution](https://github.com/heroku/heroku-buildpack-ruby/pull/892) by [@kpheasey](https://github.com/kpheasey) we can achieve the effect we want. We included that tweak on our GitHub fork of [heroku-buildpack-ruby](https://github.com/pilotcreative/heroku-buildpack-ruby/tree/feat/yarn-cache).

## Putting the solution together

#### Technical note:

> This solution works on Rails versions >= 5.1, < 6.0. It should be easy to modify it to work with other versions.

Firstly, you need to have `clean-webpack-plugin`` in place:

```
yarn add clean-webpack-plugin
```

Now add plugin's config code to `config/webpack/environment.js`:

```js
// ...
const { CleanWebpackPlugin } = require("clean-webpack-plugin");

environment.plugins.prepend("CleanWebpackPlugin", new CleanWebpackPlugin());

// ...
```

And the last step – replace the official Ruby buildpack with the custom one in your app's Settings on Heroku.

```
https://github.com/pilotcreative/heroku-buildpack-ruby.git#feat/yarn-cache
```

![Heroku buildpacks order showing Pilot's buildpack as a first one](/blog/images/faster-heroku-deploys/buildpacks.png)

Now run a full build to build your cache and you're done! For us builds take from 2.5 to 5 minutes depending on what changes are being deployed.

## Sources

- https://github.com/rails/webpacker/issues/1410
- https://github.com/heroku/heroku-buildpack-ruby/pull/892
- https://github.com/johnagan/clean-webpack-plugin
- https://github.com/rails/webpacker/pull/1744
- https://thoughtbot.com/blog/how-to-reduce-a-large-heroku-compiled-slug-size
