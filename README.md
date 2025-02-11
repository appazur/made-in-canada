# Country-Specific Static Website Content

Do you need to show small amounts of country-specific content on your website, such as prices in local currency, content related to local regulations, shipping and delivery times, or local promotional text e.g. &ldquo;🇨🇦 Made in Canada&rdquo; for example? You don't have to do it the hard way!

In these examples, perhaps the country-specific content is not important for SEO purposes.
Therefore you don't need to publish pages for each country.
Just for comparison, if country-specific SEO was important or your content was substantially different, you'd have a path or domain for mostly-duplicate content for each country e.g.

    www.example.com/us/product
    www.example.com/uk/product
    www.example.com/au/product

or 

    www.example.com/product
    www.example.co.uk/product
    www.example.co.au/product

and then use `hreflang` tags to help Google show the correct page to site visitors.

We're going to discuss a simpler approach that works with almost any web hosting platform, even if you have a simple static website with free hosting.
(For example, static websites may be hosted for free on GitHub Pages, GitLab Pages, or Google Cloud Platform.)
You don't need proprietary web hosting features or a dynamic web application server (e.g. Node.js/WordPress/Django/Rails/PHP).

### IP Geolocation API

There are a number of cloud API services that will allow you to determine a geographical location for your web visitor's IP address, such as [ipgeolocation.io](https://ipgeolocation.io), [GeoIP](https://www.maxmind.com/en/geoip-api-web-services) ($25 per 250,000 lookups) or <https://ipstack.com> ($0 - $85/mo.).

You'd then add some javascript to your site like the following (we'll take later about how this actually shows the country-specific content):

    <script src="https://.../geoip-db.min.js"></script>
    <script>
        document.addEventListener('DOMContentLoaded', function() {
            geoipDb.getCountry(function(country) {
                if (country === 'US') {
                    ...
                } else if (country === 'CA') {
                    ...
                } else {
                    ...
                }
            });
        });

This is a good approach but we chose not to use it. Considerations include:

- Does the API call have to be made for every page view, causing a delay before your country-specific content is displayed every time, or can the API response be cached by the client?
- Performance (latency) of the API service, which would affect how quickly your country-specific content is displayed.
- Dependency on 3rd-party API which may change over time, have availablity issues, or have quota limits.
- The need for an additional paid service.
- If the service is free, ensure that the included javascript file is not overly large (e.g. a large database file).

### Cloudflare IP Geolocation

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
    🇨🇦 Pricing is available in Canadian dollars.
    [Appazur](https://appazur.com/en/) is developed, supported, and hosted in Canada,
    and supports your organization in complying with Canadian privacy regulations.
    </div>
    <div class="not-canada">
    Pricing is available in US$.
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
This will work, but we chose not to use it because of the additional web request to our website, and the javascript complexity. You'd want to ensure that your AJAX code caches the response to avoid delays each time a page loads. Also ensure that this does not unintentionally inflate your page view analytics.

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
This code is free to host (up to [100,000 uses per day](https://developers.cloudflare.com/workers/platform/pricing/)) and very high performance:

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

We've presented several ways to obtain and use the country code to provide country-specific content for a static website visitor. Which approach do you prefer and why? Is there another method that you have used?
