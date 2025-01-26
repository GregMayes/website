---
title: Migrating a Jekyll Blog from jekyll-postcss to the Tailwind CLI
description: When attempting to upgrade this blog to Tailwind v4, I bumped into a couple of issues with PostCSS. This article chronicles how I removed PostCSS and migrated over to the Tailwind CLI
tags: Jekyll, CSS, Tailwind, PostCSS
---

## Background

I don't get opportunity to play around with front-end tooling much in my day-to-day work as of late. I try to use this blog as an excuse to better familiarise myself with Tailwind. I didn't know much about Jekyll when I first set up this blog, so I followed the recommended approach for installing Tailwind with Jekyll and went down the PostCSS route. I used the [jekyll-postcss](https://github.com/mhanberg/jekyll-postcss) Ruby gem to have it integrate seemlessly with Jekyll.

When I saw that [Tailwind v4 had been released](https://tailwindcss.com/blog/tailwindcss-v4), I figured I'd take the time to upgrade to it. At the time of writing, the `jekyll-postcss` gem hadn't been updated for almost 4 years. I assumed that this wouldn't be an issue because Tailwind v4 still supports PostCSS. As you might have guessed, I assumed wrong.

I bumbled along with the Tailwind v4 upgrade path. I ran the [Tailwind upgrade tool](https://tailwindcss.com/docs/upgrade-guide#using-the-upgrade-tool) (which is awesome) and it did 90% of the work, but it wouldn't touch my PostCSS configuration file as it contained "dynamic JavaScript". The upgrade guide mentions changing your postcss.config.mjs to be something like the below:

{% highlight javascript %}
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  }
};
{% endhighlight %}

There's just one problem there, my PostCSS configuration was a .js file with a CommonJS-style export statement. Before entertaining the idea of converting the file to be an .mjs file, I tried replacing `require('tailwindcss')` with `require('@tailwindcss/postcss')` in my postcss.config.js file, but I received the following error:

{% highlight bash %}
PostCSS Error!

Error: Can't resolve 'tailwindcss' in '/home/greg/Projects'
...
details: "resolve 'tailwindcss' in '/home/greg/Project Parsed request is a module"
{% endhighlight %}

The problem was likely that I was trying to require an ECMAScript module inside a CommonJS-style configuration file. I probably should have investigated it more, but I simply couldn't be bothered. I then took the logical step of trying to convert the config file to be an .mjs file but immediately ran into another issue. The `jekyll-postcss` gem only supports a postcss.config.js file and it doesn't allow for any kind of filename override.

I have an extremely low tolerance to productivity blockers. I want to be able to keep my tools on the latest versions with as little friction as possible. I don't want to clear this hurdle and then have similar problems when the next major Tailwind version is released. Dependencies are great until they're not. I made the decision to rip everything PostCSS out of my blog and replace it with the first-party Tailwind CLI. In Tailwind v3, the CLI tool is installed by default, but in v4 you have to explicitly install it.

## Removing PostCSS

Because I was using `jekyll-postcss`, when I ran `bundle exec jekyll serve` it would also start a PostCSS "server" that would react to file changes. It would be great to have this level of Jekyll integration with the Tailwind CLI, but for now, I'm content to settle for an npm command that starts a Tailwind CLI watcher.

I removed the `jekyll-postcss` gem from my Gemfile as well as any mentions of PostCSS from my _config.yml file. I also removed the `postcss`, `autoprefixer`, and `cssnano` packages from my package.json. The Tailwind CLI handles vendor prefixing and minification, so I didn't need those packages anymore.

Jekyll recommends the use of an assets directory to signify which assets (CSS, JS, images, etc.) should be brought across into the build. `jekyll-postcss` instructs you to append an empty [front matter](https://jekyllrb.com/docs/front-matter/) block at the top of your .css file(s) to denote that they should be processed by Jekyll. In my case, I had a assets/css/main.css file. I also had a syntax.css file which contained the styles for my code blocks, but I decided to fold it into main.css.

When using the Tailwind CLI, you don't need the front matter in the raw or transpiled .css file because you don't want it to be transformed by Jekyll, only moved over as is. I made a _src/css/ directory and popped my main.css file in there. I set the whole of the _src directory to be excluded by Jekyll, which means it won't end up in the built version. I also made sure to add assets/css/main.css to my .gitignore file as it will always be a build process artifact.

I created the below npm commands to make my life easier for both development and production. I thought about making a `dev` command that would run `jekyll:dev` and `tailwind:dev` concurrently, but it's not exactly arduous to open two terminal panes.

{% highlight json %}
"scripts": {
  "jekyll:dev": "bundle exec jekyll serve",
  "tailwind:dev": "npx tailwindcss -i ./_src/css/main.css -o ./assets/css/main.css --watch",
  "tailwind:prod": "npx tailwindcss -i ./_src/css/main.css -o ./assets/css/main.css --minify"
}
{% endhighlight %}

## Aftermath

Doing this migration meant that I was able to upgrade to Tailwind v4 via the migration tool, with only a couple of manual changes. I was able to drop 1 Ruby gem and 3 JavaScript packages. I did have to install the `@tailwindcss/tailwind-cli` package, but this feels like more of a technicality as it was previously included in Tailwind v3.

I don't know how useful this blog post is, but maybe it'll help other people who use Tailwind on their Jekyll blogs.