---
title: "BETA Part 1: Why I Built a Climbing Conditions Pipeline"
date: 2026-03-15
author: Tree
series: "BETA: Building a Climbing Conditions Pipeline"
tags: ["beta", "python", "github-actions", "sre", "community"]
---

Okay so lemme tell you how this started.

I was sitting in an NWAC avalanche awareness class — one my climbing community Rising Roots organized at The Mountaineers in Seattle — and I asked out loud: "how come something like NWAC avalanche forecasting doesn't exist for climbers? Some kind of tool that helps us decide when conditions are solid without having to cross-reference a shitload of resources?"

I had a quick conversation with a PhD student about weather data infrastructure, and then I sat with that question a little harder. *WTF — why doesn't this exist?*

The knowledge exists. You're checking Weather.gov here, Mountain Project forums there, texting someone who went last weekend. It's out there — but it's scattered, and a lot of it lives in informal networks. Networks that, in the PNW climbing world, have historically been pretty homogeneous and not so subtly exclusive. That accumulated knowledge about which crag dries fastest, which approach gets sketchy in March, what the river level means for the boulder field — it gets passed down. Just not always to everyone.

I'm an SRE. I build systems that take fragmented data streams and synthesize them into clear signals. The second I framed it that way I couldn't un-see it.

*This is just a conditions pipeline. I can build this.*

A week later [BETA](https://beta.trenigma.dev) was live.

---

## The stack I chose and why I'm not apologizing for it

No database. No backend. No servers.

I know, I know. But hear me out. The whole thing runs on a Python script, GitHub Actions on a 6-hour cron, and GitHub Pages. The pipeline fetches from APIs, writes a JSON file, commits it, and the static site reads from it. That's it.

I call it the logbook architecture. The pipeline writes to the logbook every 6 hours and the website reads from it. Nothing else needs to exist.

Total infrastructure cost: $0. Things that can break at 2am: basically zero. For a community tool I maintain solo, that tradeoff is a no-brainer.

---

## Four data sources, each with their own story

**Open-Meteo** was the obvious first call — free, no API key needed, covers global coordinates. One API call per crag gets me precip history, humidity, temp, wind, and 48h forecast. Easy.

**USGS stream gauges** felt like a clutch differentiator for a couple of the crags. The Skykomish River gauge at Gold Bar (`12134500`) tells me how saturated the approach terrain is near Index and Miller River. River running at 7,000+ CFS? The boulders are either underwater or the approaches are a muddy mess.

Here's where I almost lost my mind though. There's another gauge closer to Miller River (`12132000`) that shows up on the USGS map looking all active and fine. I tried to pull data from that thing and got nothing. Crickets. Turns out it's been inactive for years — just sitting there on the map like it's doing something. Had to fall back to the Gold Bar gauge as a proxy, 10 miles out but good enough. [Full writeup on the USGS integration here.](/posts/2026-03-16-beta-usgs)

**PurpleAir** was something I absolutely wanted since I've personally forgotten to check AQI before a trip more than once. I added it for wildfire smoke season — because climbing in 120 AQI air is a completely different decision than climbing in 30 AQI air. PurpleAir is crowd-sourced sensors, so coverage in the backcountry is hit or miss. My approach: bounding box query for all outdoor sensors within 30km, grab the nearest one by distance, and show the sensor name and how far it is so people know what they're actually looking at.

What I did NOT expect: on a rainy day in March, Index showed AQI 120 and Exit 38 showed AQI 115. No wildfire anywhere near Washington. Turns out cold inversion layer + everyone running their woodstove = particulates trapped at crag level. Cool insight from real data that never crossed my mind. [Full writeup on the PurpleAir integration here.](/adding-purpleair-aqi-climbing-conditions)

**NWAC** is currently link-only for the four crags with real avalanche approach terrain — Index, Miller River, Castle Rock, Leavenworth. The old API is dead and their new platform doesn't have a public API. Their data portal is restricted to approved partners, so I emailed them. In the meantime there's a "⚠️ Terrain Above" button on the relevant pages that sends you to the right forecast zone.

---

## How the go/wait/no-go signal actually works

It's a score from 0–100. Think of it like how you mentally assess conditions before a trip — multiple factors, weighted by how much each one matters.

Heavy rain in the last 24h tanks your score hard. High humidity takes a chunk. Below freezing takes another chunk. Ideal temps give you a small bonus. Incoming rain forecast docks you a bit more.

The thing I'm most proud of is the `drying_multiplier` field. Each crag has one. Peshastin Pinnacles is sandstone in an east-side rain shadow — that thing dries fast. Exit 38 is metamorphic rock in the west Cascades getting dumped on constantly — same half inch of rain hits completely differently there. The multiplier adjusts penalties accordingly.

---

## What "production-ready" actually means here

I built this the same way I'd build infrastructure at work.

Grafana Cloud monitoring with synthetic probes firing from two regions every minute. Uptime alerting to my email. Faro RUM tracking real user Web Vitals. And every pipeline run ships the full conditions payload to Loki so I can query pipeline history in Grafana.

That last one was interesting to figure out. Since the pipeline is a GitHub Actions cron job there's no persistent process to monitor. So I push the whole `conditions.json` to Loki at the end of every run. Now I can run LogQL queries over climbing conditions data. Didn't expect to be doing that this year but here we are.

There's also a history archive — every run writes a timestamped JSON snapshot. I added a "Last green" feature that scans those snapshots and surfaces when each crag last had a go signal. The question it answers: "okay but how long has it been wrecked?" Climbers ask that a lot.

---

## The part that matters most

The tech is the easy part honestly.

BETA exists because of a community conversation. It was designed around how climbers actually think. And it launched to people who immediately gave useful feedback — someone spotted on day two that the 48h forecast was influencing signals in a way that suggested trip planning use cases, not just "should I go right now." Feature direction I hadn't fully articulated yet, surfaced immediately by a user.

I'm also a co-facilitator of Rising Roots PNW — a community creating welcoming spaces for BIPOC climbers in the Pacific Northwest. When I thought about who this tool was for, I thought about that community first. Good beta means access. BETA puts it in one place, free, for anyone.

Built by a climber of color, for climbers of color — and everyone out there.

---

## What's coming

NWAC live integration when/if they respond. Trip planning view. SNOTEL snowpack data for winter approaches. A hard cap around 16 crags — this is intentionally curated, not trying to be Mountain Project.

The pipeline runs every 6 hours. Come check it out.

[BETA →](https://beta.trenigma.dev) · [GitHub →](https://github.com/trenigma/beta-trenigma-dev)

---

*Part 1 of 3 in [BETA: Building a Climbing Conditions Pipeline](/tags/beta)*
*Next: [Part 2 — Wiring in a USGS River Gauge (and the Dead One I Almost Used)](/posts/2026-03-16-beta-usgs) →*
