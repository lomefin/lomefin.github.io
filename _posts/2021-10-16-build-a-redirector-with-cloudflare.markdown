---
layout: post
title: "Build a redirector with Cloudflare"
date: 2021-10-16 17:06:35 -0400
categories:
---

# I hate Jira

But I do have to use it for some projects.

You have I hate most about it? That when you look for Jira it just takes you to the product page and there is no way to get into the project.

So basically, I've been typing all the time `clientname.atlassian.net` EVEN IF IM USING JIRA to get in.

So I wanted to make something about it.


## The rails fast approach

We have a the [Avispa.tech](https://avispa.tech) domain and I thought about making a redirector.

So firstly I used a server meant for webhooks to work on the redirections. Its a rails server so I made it like so

In routes.rb

```ruby  
get ':slug', to: 'links#redirect', constraints: { subdomain: 'redir' }
```

So if you tried `redir.yourdomain.com/jira` it would take you to your `redirects_controller` which would have something like this

{% highlight ruby linenos %}
class LinksController < ApplicationController
  def redirect
    Rails.logger.info "Redirecting to #{params[:slug]}"
    slug = params[:slug]
    return redirect_to 'https://avispa.tech' unless REDIRECTS.key? slug
    
    redirect_to REDIRECTS[slug]
  end
end
{% endhighlight %}

and REDIRECTS its just a Hash with the links hardcoded into them (we really don't get pissed with so many URLs)

So, problem solved. In 2 minutes.

## Cloudflare Workers

As I was adding the subdomain into Cloudflare, I remembered that we can have Cloudflare Workers. So why not give them a try!

So I logged into my Cloudflare account and hit the Workers option. And clicked on _Create a Worker_.

That takes me to an editor which has an example in Javascript, I switched it fast to see the [Examples](https://developers.cloudflare.com/workers/examples) so I could know how to redirect, so the first version went like this:

{% highlight javascript linenos %}
// I didn't event touch this part!
addEventListener("fetch", (event) => {
  event.respondWith(
    handleRequest(event.request).catch(
      (err) => new Response(err.stack, { status: 500 })
    )
  );
});

const REDIRECTS = {
    'jira': 'https://client.atlassian.net/jira/projects',
    'rock': 'https://web.rock.so/space/rockspace',
    'clickup': 'https://app.clickup.com/'
}

const PATHNAME_REGEX = /\/(.+)/gm;
/**
* @param {Request} request
* @returns {Promise<Response>}
*/
async function handleRequest(request) {
  const { pathname } = new URL(request.url);
  // I just extract the path
  // so something.domain.com/jira becomes '/jira'
  // so I made it 'jira'
  // It's a little overengineered but I rather use
  // regular expressions that other replacements
  const slug = pathname.replace(PATHNAME_REGEX, '$1')

  // If the slug is available, then go there, if it is missing, then avispa.tech is the fallback
  const url = REDIRECTS[slug] || 'https://avispa.tech'
  
  // Pretty straight-forward
  return Response.redirect(url, 302)
}
{% endhighlight %}

Alright, another 3 minutes, now, can it go a little more dangerous?

## The KVs

I read a long time ago along with the Workers that they were some KV stores available, KV for Key Value. So just as the Hash from before, but decoupled. So if I change the store I don't need to rewrite the Worker code.

So the example in Cloudflare Docs goes like this:

`const valueFromKV = await NAMESPACE.get('someKey')`

Alright, so what is namespace?

Went out of the editor, into the `KV` tab on top, wrote `REDIRECTS` in the namespace name, clicked on __Add__ and a row in the table below was added. 

Clicked on `View` and now I'm inside the REDIRECTS store.

Wrote down the same keys and values that I had in the code, and went back to the worker.

In my worker, I hit `Settings`, KV Namespace Bindings, Edit Binding, and I set __Variable Name__ to `REDIRECTS` and __KV Namespace__ to `REDIRECT`, I AM SO CREATIVE! Hit save and back to the editor!

So now I removed the REDIRECTS Hash from the code and switched __handleRequest__ into this

{% highlight js linenos %}
async function handleRequest(request) {
  const { pathname } = new URL(request.url);
  // The substituted value will be contained in the result variable
  const slug = pathname.replace(PATHNAME_REGEX, '$1')
  const url = await REDIRECTS.get(slug) || 'https://avispa.tech'
  
  return Response.redirect(url, 302)
}
{% endhighlight %}

And that's it!

That's it? No, not yet. That worker is working in its very custom subdomain, but not the one that we want. Here is where I'm really not sure what I'm doing, but made it work anyhow.

Now I went back to the domain where I want to attach the worker to, clicked the __Workers__ tab, __Add route__, set __Route__ as `redirects.mydomain.com/*` and __Worker__ as the one that I just created (in my case `redirector`). Saved it, aaaand... nothing.

So I just went crazy about it, disclaimer: I didn't even wait for 2 seconds, and I headed to __DNS__ tab, and added a `CNAME` row with `redirects.mydomain.com` headed to anywhere.

After saving that new row and waiting 10 seconds... Everything was working!

So, yeah, 5 minutes later, the KV Backed-Cloudflare Worker run-URL redirector of my own is up and running, I wish I had read more about how Workers are __attached?__ would have been productive to understand the last step, but the experience was butter-smooth in my opinion, give Cloudflare workers a try!

