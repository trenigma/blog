---
title: "Building a Personal NOC: I Needed to Know If Anyone Actually Uses My Shit"
date: 2026-03-17
draft: false
tags: ["monitoring", "observability", "grafana", "faro", "synthetic-monitoring", "side-projects"]
description: "How I set up production-grade monitoring for my three side projects (BETA, blog, littlelink) using Grafana Cloud's free tier, and why I now have a spare TV running my personal NOC dashboard 24/7."
---

I built three sites. All on GitHub Pages. All costing $0 to run. And I had absolutely no idea if anyone was actually using them.

- **BETA** (beta.trenigma.dev) - Cascade climbing conditions tool
- **Blog** (blog.trenigma.dev) - This thing you're reading
- **littlelink** (littlelink.trenigma.dev) - Link-in-bio page

They all worked. GitHub Actions deployed them. DNS resolved. SSL certs were valid. Cool.

But like... were people visiting? Which pages? From where? Was the site actually fast or just fast for me on my gigabit connection in Seattle?

I had no fuckin clue.

## Love Me Some $Free.99

I'd been meaning to set up monitoring for a while. But every time I looked into it, the options were:
- Self-hosted Prometheus/Grafana (nope, just escaped that hell with the Ghost migration)
- Datadog ($$$)
- New Relic ($$$)
- Random SaaS tools that want $50/month for basic metrics

Then I actually read the Grafana Cloud free tier limits:
- 50GB of logs
- 10k series for metrics
- 500 VUh (Virtual User hours) for synthetic monitoring
- 50GB of traces

For free. Forever.

For three small side projects? That's basically infinite.

## The Goal

I wanted to know:
1. **Is the site up?** (uptime monitoring)
2. **How fast is it actually loading?** (real user metrics)
3. **Where are people coming from?** (geographic data)
4. **What are they actually looking at?** (page views)
5. **Which buttons are people clicking on littlelink?** (dopamine)

And I wanted all of this in a single pane of glass.

Because why not.

## Setting Up Grafana Cloud

**Step 1: Create account**

Go to grafana.com, sign up for free tier. They don't even ask for a credit card. Respect.

**Step 2: Create a stack**

Give it a name (I went with `betatrenigma` because BETA was the first thing I wanted to monitor). Pick a region close to you.

You get:
- Prometheus endpoint (for metrics)
- Loki endpoint (for logs)
- Tempo endpoint (for traces)
- Synthetic Monitoring (for uptime checks)
- Frontend Observability (for RUM)

All preconfigured. Just works.

## Synthetic Monitoring: Is It Actually Up?

First priority: uptime checks. I want to know if my sites go down before someone tells me on Mastodon.

**In Grafana Cloud:**
1. Go to **Synthetic Monitoring**
2. Click **Add new check**
3. Configure:
   - **Job name:** `beta-trenigma-dev`
   - **Target:** `https://beta.trenigma.dev`
   - **Check type:** HTTP
   - **Probe locations:** London, North Virginia, Oregon (global coverage!)
   - **Frequency:** 60s
   - **Timeout:** 10s

Hit save. Repeat for blog and littlelink.

Now I've got:
- 3 sites monitored from 3 locations globally
- Checks every minute
- Automatic alerting if anything goes down
- Response time data from different regions

Total setup time: 5 minutes.

And here's the thing: **GitHub Pages uptime is stupidly good**. All three sites? 100% uptime. Always green. North Virginia responds in ~24ms, Oregon in ~50ms, London in ~100ms.

It's almost boring. But boring is good.

## Faro RUM: Real User Monitoring

Synthetic checks tell you if the site is up. But they don't tell you how fast it actually loads for real people on real connections.

Enter Faro: Grafana's Real User Monitoring SDK.

**For each site, I created a Faro app:**

In Grafana Cloud → **Frontend Observability** → **Connect new app**

- App name: `beta-trenigma-dev` (or `trenigma-blog`, `littlelink-trenigma-dev`)
- Domain: `beta.trenigma.dev`

This gives you a collector URL and a script snippet.

**Add the snippet to your site's `<head>`:**

```html
<script>
(function () {
  var webSdkScript = document.createElement("script");
  
  webSdkScript.src =
    "https://unpkg.com/@grafana/faro-web-sdk@2/dist/bundle/faro-web-sdk.iife.js";
  
  webSdkScript.onload = () => {
    window.GrafanaFaroWebSdk.initializeFaro({
      url: "https://faro-collector-prod-us-west-0.grafana.net/collect/YOUR_ID",
      app: {
        name: "beta-trenigma-dev",
        version: "1.0.0",
        environment: "production",
      },
    });
  };
  
  document.head.appendChild(webSdkScript);
})();
</script>
```

Commit. Push. Wait 5 minutes.

**Now I can see:**
- Page views over time
- Unique sessions
- Performance metrics (TTFB, FCP, LCP)
- Which pages people actually visit
- Geographic distribution
- Browser/device breakdown

And the data is actually interesting:
- BETA gets way more traffic than my blog (climbers > SREs, apparently)
- Most visitors are from Seattle/Portland (PNW climbing community, makes sense)
- Blog posts load in ~300ms (TTFB), which is solid for a static site
- People actually read the whole post (scroll depth tracking)

## The Dopamine Metric: Button Click Tracking

For littlelink, I wanted to know which buttons people actually click.

Like, are people going to my blog? BETA? Ko-fi? (lol, probably not Ko-fi)

Faro lets you send custom events. So I added this to littlelink:

```javascript
// Track button clicks
window.addEventListener('load', () => {
  setTimeout(() => {
    if (window.GrafanaFaroWebSdk && window.GrafanaFaroWebSdk.faro) {
      const faro = window.GrafanaFaroWebSdk.faro;
      
      document.querySelectorAll('.link-button').forEach(button => {
        button.addEventListener('click', (e) => {
          const buttonName = e.currentTarget.querySelector('span').textContent;
          const buttonUrl = e.currentTarget.getAttribute('href');
          
          faro.api.pushEvent('button_click', {
            button_name: buttonName,
            button_url: buttonUrl
          });
          
          console.log('Tracked click:', buttonName);
        });
      });
    }
  }, 1000);
});
```

Now every button click sends an event to Grafana with:
- Which button was clicked
- Where it goes
- Timestamp

And I can build charts showing:
- Total clicks
- Top clicked buttons (bar chart)
- Clicks over time

**Early results:** BETA button gets clicked the most. Blog is second. Ko-fi... yeah, nobody clicks Ko-fi. But that's fine. It's there.

This is the **dopamine metric**. Watching that click counter go up? Feels good.

## Building the Unified Dashboard

Now I've got data. Time to make it actually useful.

**Goal:** One dashboard. All three sites. Single pane of glass.

I built a dashboard called `trenigma-sites` with 8 rows:

**Row 1: Overview**
- Overall uptime (combined across all 3 sites)
- Average response time
- Response time trends

**Row 2: BETA**
- Uptime percentage
- Response time by probe location

**Row 3: Blog**
- Uptime percentage
- Response time by probe location

**Row 4: Geographic Breakdown**
- Response time by location (table)
- Checks by probe location (pie chart)

**Row 5: littlelink**
- Uptime percentage
- Response time by probe location

**Row 6: RUM - Page Views**
- Total page views (all sites)
- Page views over time (by site)

**Row 7: RUM - Performance Metrics**
- TTFB (Time to First Byte)
- FCP (First Contentful Paint)
- LCP (Largest Contentful Paint)

**Row 8: littlelink Button Clicks**
- Total clicks
- Top clicked buttons (bar chart)
- Clicks over time

All the queries are PromQL hitting the Grafana Cloud Prometheus endpoint. Faro data flows through the same system.

**Auto-refresh:** 30 seconds.

Is it necessary? Absolutely not.

Is it cool as fuck? Yes.

Does it make me feel like I'm running a real operation instead of three side projects on GitHub Pages? Also yes.

## What This Actually Costs

**Grafana Cloud free tier:**
- Cost: $0/month
- Limits: Way more than I'll ever hit
- Features: Everything I need

**Total monthly cost for production-grade monitoring: $0**

Compare that to Datadog at $15/host or New Relic at $99/month.

## What I Actually Learned

**1. (Some) Free tiers are underrated**

Grafana Cloud's free tier is genuinely good. Not "free but useless." Actually good.

**2. Monitoring changes behavior**

Once I could see page views, I started caring more about writing. Seeing the click counts on littlelink made me actually update it.

Metrics create accountability. Even if it's just to yourself.

**3. GitHub Pages is fast**

Like, really fast. 100% uptime, sub-100ms response times globally. For free. Wild.

**4. People actually use my stuff**

BETA is getting page views which feels so good. Blog posts get read. People click through from littlelink.

It's not huge numbers but they're real people. And knowing that feels good.

## Setting This Up Yourself

If you want to replicate this:

**1. Sign up for Grafana Cloud** (free tier)

**2. Add Faro RUM to your sites:**
- Create a Faro app in Frontend Observability
- Add the script snippet to your `<head>`
- Deploy

**3. Set up Synthetic Monitoring:**
- Create HTTP checks for each site
- Pick 3 probe locations
- Set frequency to 60s

**4. Build a dashboard:**
- Create panels for uptime, response time, page views
- Use PromQL queries for metrics
- Set auto-refresh to 30s

**5. (Optional) Set up a TV NOC:**
- Get a Raspberry Pi
- Install Chromium
- Set it to kiosk mode loading your dashboard
- Mount TV on wall
- Feel like a sysadmin from 2010

## What's Next

Now that I have monitoring, I can actually make data-driven decisions:

- Which BETA crags get the most traffic? → Prioritize those for feature development
- Which blog posts get read? → Write more like those
- What time of day do people visit? → Maybe schedule posts better

Plus I want to add:
- Alerting (Slack notifications if a site goes down)
- More custom events (track specific user flows)
- Definitely need to add those hero images to blog posts eventually

But the core is done. Three sites, full monitoring, one dashboard, $0/month.

---

**Current stack:**
- Grafana Cloud (RUM + Synthetic Monitoring)
- Faro Web SDK (client-side instrumentation)
- GitHub Pages (hosting)

**Monthly cost:** ~$0
**Monitoring coverage:** 100%  
**Dopamine from watching metrics:** Priceless

If you're running side projects and have no idea if anyone uses them, set up monitoring. It takes an hour. It costs nothing. And you'll actually know if your work matters.

Now go build your personal NOC. You know you want to. 📊🔥