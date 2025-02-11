# Country-Specific Static Website Content

Want to show country-specific content on your website, such as prices, or &ldquo;🇨🇦 Made in Canada&rdquo; for example?
This is easy to do on any website, even if you have a simple static website with free hosting.
(For example, static websites may be hosted for free on GitHub Pages, GitLab Pages, or Google Cloud Platform.)
You don't need proprietary web hosting features or a dynamic web application server (e.g. Node.js/WordPress/Django/Rails/PHP).

### IP Geolocation

There are a number of cloud API services that will allow you to determine a geographical location for your web visitor's IP address, such as <https://ipstack.com>.
We chose Cloudflare IP Geolocation to avoid the need to call an API from Javascript after the page loads.
Rather, the geolocation information is provided to you immediately as a header of your initial web page.

The Cloudflare proxy with IP Geolocation is available with Cloudflare's free tier, and works with your custom domain name and static hosting provider.
Your website may already have this, as "Cloudflare is used by around 19.3% of all websites on the Internet for its web security services, [as of January 2025](https://en.wikipedia.org/wiki/Cloudflare)".

When you use Cloudflare in front of your website, you can simply toggle on the **IP Geolocation** feature.

> IP geolocation adds the `CF-IPCountry` header to all requests to your origin server.
> Cloudflare automatically updates its IP geolocation database using MaxMind and other data sources, typically twice a week.
> Reference: <https://developers.cloudflare.com/network/ip-geolocation/>

![Cloudflare](https://developers.cloudflare.com/_astro/logo.p_ySeMR1.svg)

### CSS

Since our goal is to show country-specific content, we can use CSS classes to mark HTML div's as in this example:

    <div class="canada">
    🇨🇦 Buy Canadian! We offer pricing in $CAD. [Appazur](https://appazur.com/en/) is developed, supported, and hosted in Canada.
    </div>

By default, we'll hide the country-specific content using our CSS stylesheet:

    .canada, .not-canada {
        display: none;
    }

### Accessing the `CF-IPCountry` Header

Since we're not relying on a dynamic web application server, this CSS is static.
We'll need to use client-side Javascript to modify the visibility of these page elements.
However, due to the browser security model, Javascript cannot access the current page's HTTP response headers, so a workaround is necessary in order to proceed.

#### AJAX

One possible approach is to make a second web request to our web page from Javascipt, using AJAX i.e. `fetch()` or `XMLHttpRequest()`.
(If you're not familiar with AJAX, that's OK - the proposed solution will not use it.)
Such an AJAX request, initiated from Javascript, [does allow for inspecting the response headers](https://stackoverflow.com/questions/220231/accessing-the-web-pages-http-headers-in-javascript).
This will work, but we chose not to use it because of the additional web request to our website, and the javascript complexity. You'd want to ensure that your AJAX code caches the response to avoid delays each time a page loads, and additional load on the web server.

    const resp = await fetch(document.location.href, {method: 'HEAD'});
    const headers = Object.fromEntries(resp.headers.entries());

Without AJAX, and with a static website, is there another option?
Let's take a moment to consider what we could do IF we did have a dynamic web application server.

#### Web Application Server

We could create an endpoint like the following (Python/Django):

    class DetectIpCountryView(View):
        @method_decorator(cache_control(private=True, max_age=432000))
        def get(self, request, *args, **kwargs):
            return HttpResponse('''cf_ipcountry="{}";
    '''.format(request.META.get('HTTP_CF_IPCOUNTRY', '')))

Then in our HTML we can include the above endpoint like this:

    <script src="https://my-django-server/geoip.js"></script>

which effectively injects a javascript snippet like this into our web page:

    cf_ipcountry="US";

As with the rejected AJAX approach, this does involve an extra web request, but it is cacheable so this will only occur on the first access to any of your pages.

Great! But we didn't want this web application server dependency.

#### Serverless Worker

Since we are already dependent on Cloudflare, we also have access to free [Cloudflare Workers](https://developers.cloudflare.com/workers/).
With Cloudflare Workers, we can create a high performance cloud endpoint that performs the same function as the above.
The code is not quite as concise, but it will be free to host and very high performance:

    addEventListener('fetch', event => {
        event.respondWith(handleRequest(event.request))
    })

    async function handleRequest(request) {
        return res = new Response(
            'cf_ipcountry="' 
                + request.headers.get('cf-ipcountry')
                + '";',
            {
                status: 200,
                headers: {
                    'cache-control': 'private, max-age=432000',
                    'Content-Type': 'text/javascript'
                }
            }
        );
    }

You can then publish it and include it in your HTML like this:

    <script src="https://geoip.my-cloudflare-account.workers.dev/"></script>

### Finally, the Javascript

Now it is trivial to show the country-specific content based on the country code for the web visitor:

    let country_code = window.cf_ipcountry;
    console.log('Country:', country_code);
    let els = document.querySelectorAll(country_code == 'CA' ? ".canada" : ".not-canada");
    let i;
    for (i = 0; i < els.length; i++) {
        els[i].style.display = 'block';
    }

### Example Sites and Source Code

You can see this approach in use on these websites:

- [appazur.com](https://www.appazur.com/en/?ref=madeincanada)
- [halltrak.app](https://www.halltrak.app/en/?ref=madeincanada)

And the source code referenced above is freely available here:

- https://github.com/appazur/made-in-canada

We've presented several ways to obtain and use the country code for a static website visitor. Which approach do you prefer and why?
