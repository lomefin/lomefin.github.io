---
layout: post
title: "Using importmap in Ruby on Rails"
date: 2022-01-24 3:14:35 -0400
categories:
---

# I'm a big fan of software packaging

But sometimes you are building a site that really doesn't use code, specially javascript,
that is *that* complicated, so why lose the ability to have all those libraries already 
available on the internet?

What's cool about having them published already? First, you don't have to compile just another 
version that looks the same as the original, so no wasting resources, second, you don't force 
your user to download that custom-but-really-not-custom file with the code, maybe the use already 
has the file in its cache, so no need to load it, even less waste.

So, I built a site called [dolarapesos.cl](https://dolarapesos.cl) which gets the exchange rate between 
the Chilean Peso and the US Dollar every day and publishes it. It uses a Rails App and for the first time 
it uses an importmap. In my own words, importmap is a technique to declare external JS dependencies on the 
internet so there is no need to have them compiled locally.

You can read a lot about it in the Rails site, but I will just write down the shortcut version.

If you have a library you need to import, say [ChartJS](https://www.chartjs.org/) into your code, just type 
down in your rails directory `bin/importmap pin chartjs`. That command will resolve the URL of the latest version 
(by default it looks on jspm.io) and will write down into your `importmap.rb` file a command to `pin` the library
to an URL.

My `importmap.rb` looks like this at this moment:

```ruby
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
pin "@hotwired/stimulus-loading", to: "stimulus-loading.js", preload: true
pin_all_from "app/javascript/controllers", under: "controllers"
pin "bootstrap", to: "https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.esm.js"
pin "@popperjs/core", to: "https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.0/lib/index.js"
pin "chart.js", to: "https://ga.jspm.io/npm:chart.js@3.7.0/dist/chart.esm.js"
pin "lazysizes", to: "https://ga.jspm.io/npm:lazysizes@5.3.2/lazysizes.js"
```
Cool, huh? Now we have our dependencies declared, so how do we use it?

Well.. we just use it. For example, where I use ChartJS (it is an Stimulus Controller) I can use it like this:

```javascript
import { Controller } from "@hotwired/stimulus"
import { Chart, registerables  } from "chart.js"
Chart.register(...registerables )
```

Did you get it? With the newest versions of javascript supported by the browser we can import from an URL, but with importmap 
we make an index saying `chart.js` translates into `https://ga.jspm.io/npm:chart.js@3.7.0/dist/chart.esm.js` so instead of doing
`import { Chart, registerables  } from "https://ga.jspm.io/npm:chart.js@3.7.0/dist/chart.esm.js"` we just name it `chart.js`

I have to declare that I still get confused with modules and libraries, but I can understand that those `.esm.js` files are importable 
modules, like ChartJS, and lazysizes is just a library that gets imported right away into the document, so those are instantly available.


