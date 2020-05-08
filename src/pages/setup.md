---
title: 'Site Setup'
permalink: '/setup/index.html'
introSummary: 'How the sausage is made...'
---
I've been a big fan of static site generators for a long time, and I'm a well known fan of [Node js](https://nodejs.org/en/), which makes [Eleventy](https://www.11ty.dev) the no-brainer for me. It can be as simple or as complex as you want, but for the moment simplicity was the key.

To that end, I looked around for some ready made blog templates that looks good, and settled on [Hylia](https://hylia.website). Normally I take a template and hack it all to pieces to try and get something unique, losing momentum halfway through and giving up -littering my drives with partial sites and grand plans. Not this time. What you see is pretty bog standard, with a few small tweaks here and there.

The site code is all [stored on Github](https://github.com/tinyexplosions/tinyexplosions.com/), and posts are all made by creating markdown files, with a little sprinkle of YAML for frontmatter. I don't need any kind of CMS or GUI, so it works well for me - I can draft anywhere, on mobile or desktop etc, and just makes it easier for me.

Hylia is also **really** useful, in that it has an easy integration with [Netlify](https://www.netlify.com) - in fact, if you start at [hylia.website](https://hylia.website), and follow the "deploying Hylia to Netlify." it will take you through linking Netlify to your GitHUb account, will create an appropriate repository and hook into Netlify's CD pipleine so that any push to the repo will rebuild your site.

If you're unbothered by custom domains, you can stop there and use the Netlify provided one ([frosty-cray-edb44d.netlify.app](https://frosty-cray-edb44d.netlify.app) is this site), or you can go to a friendly domain register like [Hover](https://www.hover.com/), and connect a domain to Netlify - they'll even handle setting up LetsEncrypt for SSL certs!

If JavaScript isn’t your cup of tea (though it really should be), similar setups exist in other languages, for example [Hugo](https://templates.netlify.com/tags/hugo/) -written in Go, [Gatsby](https://templates.netlify.com/tags/gatsby) -JavaScript, specifically React, if python is your go to, there’s [Hyde](https://templates.netlify.com/tags/hyde) and if you still like Ruby, there’s [Jekyll](https://templates.netlify.com/tags/jekyll). 

It’s also worth mentioning that for everything outlined, the only cost is that of a domain name, so the barrier to entry is incredibly small - you can even get setup and play around entirely for free, and add in a domain later on, so there’s really no excuse not to have a play.