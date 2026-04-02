---
title: "I Added Sun Shading to My Climbing Conditions Tool Using Nothing but Math"
date: 2026-04-01
draft: false
tags: ["BETA", "javascript", "solar-geometry", "frontend"]
description: "How I built real-time sun/shade awareness for 12 PNW crags with zero dependencies, zero API calls, and about 330 lines of JavaScript."
hero_image: "/images/hero/smith-rock-sun-shade.jpg"
hero_alt: "Afternoon sun at Smith"
---

During a recent conversation with a fellow climber, I was introduced to the idea of sun shading as a useful thing to have in BETA. Which walls are in sun right now, which are in shade, when does it flip.

I noodled on it, then decided to see if I could get it built.

## The problem is actually two questions

"Is this wall in the sun?" sounds simple. But it breaks down into two separate things you need to know:

1. Where is the sun right now? (compass direction and how high above the horizon)
2. Which direction does the wall face? (its outward-facing bearing)

If you know both, the rest is just angle math. Stand at the base of a wall and look straight out. If the sun is anywhere in your 180-degree field of view, the wall is getting hit. That's the entire concept.

## Solar position is deterministic

You don't need an API for sun position, which is nice. Given a latitude, longitude, date, and time, where the sun sits in the sky is pure math. Astronomers have had this figured out for centuries.

The algorithm comes from [SunCalc](https://github.com/mourner/suncalc) by Vladimir Agafonkin (the same person behind Leaflet). It works like this:

1. Convert the current date/time to Julian days since J2000 (January 1, 2000 at noon)
2. Calculate the sun's ecliptic longitude using Kepler's equation
3. Convert ecliptic coordinates to equatorial coordinates (right ascension and declination)
4. Factor in the observer's latitude and local sidereal time
5. Output: **azimuth** (compass bearing) and **altitude** (degrees above horizon)

I pulled the core math into my own file rather than importing the full library. The key function:

```javascript
function getSunPosition(date, lat, lng) {
  var lw = -lng * RAD;
  var phi = lat * RAD;
  var d = toDays(date);
  var c = sunCoords(d);
  var H = siderealTime(d, lw) - c.ra;

  var altitude = Math.asin(
    Math.sin(phi) * Math.sin(c.dec) +
    Math.cos(phi) * Math.cos(c.dec) * Math.cos(H)
  );
  var azimuth = Math.atan2(
    Math.sin(H),
    Math.cos(H) * Math.sin(phi) - Math.tan(c.dec) * Math.cos(phi)
  );

  return {
    altitude: altitude / RAD,
    bearing: (azimuth / RAD + 180 + 360) % 360
  };
}
```

Just trigonometry that runs in microseconds on any device.

## Wall aspect as a bearing

Every cliff face points in a direction. Index Town Wall faces south. Frenchman Coulee faces west. The Stawamus Chief faces west. I already had aspect data in `crag-meta.js` (the single source of truth for all crag metadata in BETA) as cardinal directions. Converting those to numeric bearings was a one-liner per crag:

```javascript
// crag-meta.js
'index': {
  lat: 47.8248, lng: -121.5595,
  aspect: 'South', sun_bearing: 180,
  // ...
},
'exit-38': {
  lat: 47.4425, lng: -121.7026,
  aspect: 'Southeast', sun_bearing: 135,
  // ...
},
```

The `sun_bearing` field is the compass direction the wall faces outward. South = 180°. Southwest = 225°. West = 270°. You get it.

## The sun/shade check is four lines

Once you have the sun's bearing, the sun's altitude, and the wall's bearing, determining sun or shade looks like:

```javascript
function isWallInSun(sunBearing, sunAltitude, wallBearing) {
  if (sunAltitude <= 0) return false;
  var delta = Math.abs(sunBearing - wallBearing);
  if (delta > 180) delta = 360 - delta;
  return delta < 90;
}
```

Sun below the horizon? Shade. Sun more than 90 degrees away from the wall's facing direction? Shade. Otherwise: sun.

I validated this against physical intuition for three crags:

- **Index (South, 180°)**: Sun at 8 AM is bearing ~96° (east). Delta from 180° = 84°. Under 90, so in sun. Correct. South-facing walls catch morning sun in the PNW because the sun rises in the east-southeast and sweeps south.
- **Vantage (West, 270°)**: Sun at noon is bearing ~157° (south-southeast). Delta from 270° = 113°. Over 90, so in shade. Correct. West-facing basalt columns don't see sun until mid-afternoon.
- **Exit 38 (Southeast, 135°)**: Sun at 3 PM is bearing ~217° (southwest). Delta from 135° = 82°. Under 90, so still catching the tail end of sun. Correct but marginal. The walls lose direct sun shortly after.

Every result matched what you'd expect if you've climbed these places.

## Finding the transition time

Knowing "in sun" or "in shade" right now is useful. Knowing *when it flips* is more useful. A climber checking conditions at 2 PM wants to know if the wall they're driving to will still have sun when they arrive at 4 PM.

I step forward in 5-minute increments and check when the boolean flips:

```javascript
function findNextTransition(now, lat, lng, wallBearing, currentlyInSun) {
  var STEP = 5 * 60 * 1000; // 5 minutes
  var t = now.getTime();

  for (var i = 0; i < 288; i++) { // 24 hours max
    t += STEP;
    var pos = getSunPosition(new Date(t), lat, lng);
    var inSun = isWallInSun(pos.bearing, pos.altitude, wallBearing);
    if (inSun !== currentlyInSun) return new Date(t);
  }
  return null;
}
```

288 iterations of lightweight trig. Runs in single-digit milliseconds. The UI shows something like:

> ☀️ South-facing wall currently **in sun**
> Enters shade around 6:54 PM
> Sunset around 7:34 PM

## The "mixed aspect" problem

Seven of my twelve crags have a single dominant wall direction. Those get the full sun/shade verdict. But five crags (Leavenworth, Mt. Erie, Smith Rock, Smoke Bluffs, Miller River) have walls facing every which way. Telling someone "Leavenworth is in the sun" is meaningless when Castle Rock faces south and Icicle Creek Buttress faces north.

For now, those crags get a different display:

> ☀️ Sun is to the **WSW** at 26° elevation
> Sunset around 7:31 PM

If you know the crag, this is immediately useful. You know which sectors face which direction. The sun position tells you everything you need to fill in the blanks.

Phase 1.5 will add named sectors with individual bearings to `crag-meta.js`, so Smith Rock could show "Morning Rock: in shade / Dihedrals: in sun" separately. The solar math doesn't change at all. It's purely a data curation project.

## Architectural decisions

A few choices I made and why:

**Client-side, not pipeline.** BETA's conditions data runs through a Python pipeline on a 6-hour cron via GitHub Actions. Sun position changes by the minute, so baking it into a 6-hour snapshot makes no sense. It has to be real-time, which means client-side JavaScript. The sun position at 2 PM is different from 2:05 PM.

**Inlined math, not a library import.** SunCalc is only ~3KB, but I extracted just the position calculation (~40 lines) rather than importing the full package. BETA has zero runtime dependencies and I want to keep it that way. The math is stable (the Earth's orbit isn't getting patched) so there's no maintenance cost to inlining it.

**5-minute refresh interval.** The widget recalculates every 5 minutes. Sun position at PNW latitudes changes slowly enough that 5 minutes is more than sufficient. No visible jumpiness, minimal CPU.

**Bearing tolerance of 90°.** The "is the sun hitting this wall" check uses a 90-degree half-angle. In reality, low-angle sun from far off to the side barely illuminates a wall. I could tighten this to 75° or 80° for a more conservative "meaningful sun" threshold. But 90° matches the physical intuition of "can you see the sun from the base of the wall" and that felt right for v1.

## What Phase 2 looks like

The elephant in the room is terrain. A south-facing wall might theoretically see the sun at 8 AM, but if there's a ridge across the valley casting a shadow until 10 AM, the math is wrong. This is especially relevant for crags like Index, where the Skykomish valley walls block early morning sun, and Gold Bar, where the narrow canyon geometry creates late morning shadows.

Phase 2 would use USGS Digital Elevation Models (DEMs) to calculate viewshed/hillshade for each crag. Given the wall position and a DEM tile, you can trace rays from the sun's position and check if terrain blocks the line of sight. USGS has free 10-meter resolution DEMs for the entire Cascades.

The math is heavier (ray-terrain intersection for each check) but still deterministic and still client-side viable. The DEM tiles could be pre-processed into a compact horizon profile per crag: for each compass bearing, what's the minimum sun altitude needed to clear the terrain. That turns a 3D ray-tracing problem into a simple lookup table.

That's a bigger lift, but the foundation is in place.

## The whole thing is ~330 lines

No build step. No bundler. No framework. One JavaScript file that does trig, checks angles, and writes some HTML into a div. The data lives in the same `crag-meta.js` file that already powers every crag detail page.

Check it out live: [beta.trenigma.dev](https://beta.trenigma.dev)

Source: [github.com/trenigma/beta-trenigma-dev](https://github.com/trenigma/beta-trenigma-dev)