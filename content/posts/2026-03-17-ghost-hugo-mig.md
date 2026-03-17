---
title: "Ghost → Hugo: My EC2 Box Died and I'm Not Going Back"
date: 2026-03-17
draft: false
tags: ["infrastructure", "hugo", "ghost", "migration", "aws", "github-pages", "cost-optimization"]
description: "How my Ghost blog on EC2 became too broken to even SSH into, forcing me to migrate to Hugo via Apple Notes backups. Also: I'm never paying AWS $400/year for a blog again."
---

So my Ghost blog died. Not like gracefully-sunsetting died. Like *can't SSH in anymore* died.

I'd been running Ghost on an EC2 instance for about a year. Docker, Traefik, MariaDB, the whole self-hosted stack. It was beautiful when it worked. When it didn't work, I'd spend Friday nights debugging why the database ate all the RAM again or why SSL certs didn't renew properly.

The Free Tier year ended. AWS started charging me. And the instance just... got slower. And slower. Until one day I tried to SSH in and it just hung there. Connection timeout. 

I could've debugged it. Spun up a new instance, attached the EBS volume, forensics'd the thing. But honestly? I looked at my AWS bill (~$35/month for a blog that maybe 100 people read) and thought: *why am I doing this to myself?*

## The Apple Notes Rescue Mission

Your backups are only as good as your ability to access them when shit hits the fan.

I had database backups: automated MariaDB dumps to S3. But to restore them, I'd need... a working database instance. Which meant spinning up more AWS infrastructure. Which meant more money. For a blog.

You know what I did have? **Apple Notes.**

Turns out, past-me had been drafting posts in Apple Notes before publishing them to Ghost. Not all of them. But the ones I actually cared about? Yeah, those were there.

So my migration strategy became: fuck it, start fresh. Copy the posts I care about from Apple Notes, leave the rest in the Ghost dumpster fire, move on.

## Why Hugo (and Why Now)

I needed something that:
1. Costs $0
2. Doesn't require SSH access to deploy
3. Won't die if I ignore it for 6 months
4. Lets me write in Markdown (because I was already doing that anyway)

Hugo checked all those boxes. Static site generator, builds in milliseconds, deploys via GitHub Actions. No database. No server. No Docker containers quietly eating RAM at 3am.

Hugo vs Jekyll vs Eleventy? I went with Hugo because it's fast and I'd seen it before. Didn't overthink it.

## The Actual Migration

**Step 1: Install Hugo**

```bash
brew install hugo
hugo new site blog
cd blog
```

**Step 2: Pick a theme**

I went to [themes.gohugo.io](https://themes.gohugo.io), found one that looked simple and cloned it.
```bash
git init
git submodule add https://github.com/THEME/REPO themes/theme-name
```

Then spent 20 minutes tweaking `config.toml` until it didn't look broken.

**Step 3: Migrate content (the messy way)**

Opened Apple Notes. Opened Hugo. Copy-pasted.

For each post:

```bash
hugo new posts/post-slug.md
```

Then manually copy the Markdown from Apple Notes into the new file, add frontmatter:

```yaml
---
title: "Whatever The Post Was Called"
date: 2025-12-whatever
tags: ["infrastructure", "debugging"]
---

[paste content here]
```

Repeat 10 times. Realize I only really care about 10 posts anyway. Ship it.

**Step 4: Images? LOL**

My Ghost blog had images. Beautiful Unsplash hero images on every post. Made it look professional.

Did I migrate those? Nope.

Did I figure out how to make Hugo look that pretty? Also nope.

I was in MVP mode. Get the words onto the internet. Make it pretty later (spoiler: I still haven't).

Images can wait. Shipping > perfection.

**Step 5: GitHub Actions for deploy**

Created `.github/workflows/deploy.yml`:

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [main]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true
          
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          
      - name: Build
        run: hugo --minify
        
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

Commit. Push. Watch GitHub Actions do its thing. Site goes live.

No SSH. No Docker. No database dumps. Just `git push`.

## The DNS Bullshit

Of course it wasn't smooth. DNS is never smooth.

**Problem 1:** I created a `CNAME` file in the repo root with `blog.trenigma.dev` in it. Pushed it. GitHub Actions built the site. The `CNAME` file disappeared.

Turns out Hugo only deploys what's in the `public/` directory after build. Random files in the repo root? Ignored.

**Fix:**

```bash
echo "blog.trenigma.dev" > static/CNAME
```

Now it gets included in the build output.

**Problem 2:** I had GitHub Pages set to "Deploy from a branch" initially. Then switched to "GitHub Actions" to use the workflow.

Here's the fun part: when you switch deployment methods, **GitHub silently clears your custom domain field**. No warning. Bitches.

So I'm sitting there watching my GitHub Actions succeed, visiting `blog.trenigma.dev`, getting 404s, checking DNS records 40 times...

The fix? Go to Settings → Pages, re-enter `blog.trenigma.dev` in the custom domain field, click Save.

Then wait 10 minutes for GitHub to verify DNS and provision SSL certs. Touch grass. Come back. It works.

## Killing the EC2 Instance

Once the Hugo blog was live, I had one task left: murder the old infrastructure.

**First: backups** (even though I couldn't SSH in)

```bash
# Create AMI snapshot just in case
aws ec2 create-image \
  --instance-id i-xxxxxx \
  --name "ghost-blog-backup-$(date +%Y%m%d)" \
  --description "Backup before termination"
```

If I ever need those old posts that aren't in Apple Notes? They're in the AMI. Will I ever actually restore it? Probably not. But it exists.

**Then: terminate**

```bash
aws ec2 terminate-instances --instance-ids i-xxxxxx
```

**Release the Elastic IP** (you get charged for unused ones):

```bash
aws ec2 release-address --allocation-id eipalloc-xxxxxx
```

**Delete orphaned EBS volumes.**

Done. Infrastructure: deleted. Monthly AWS bill: $0.

## What This Actually Feels Like

**Before:**
- Monthly cost: ~$35
- Deploy process: SSH into EC2, `docker-compose pull`, restart services, pray
- SSL renewal: Usually works, sometimes doesn't, always make me wanna throw shit
- Backups: Scripted MariaDB dumps to S3 that I hope work but have never tested
- Monitoring: "Is the site up? Let me check.... Fuck."

**After:**
- Monthly cost: $0
- Deploy process: `git push`
- SSL: GitHub handles it
- Backups: The entire site is in Git. If GitHub goes down, we have bigger problems.
- Monitoring: GitHub tells me if Actions fail. Otherwise I don't care.

Annual savings: ~$420.

That's a weekend climbing trip to Squamish. That's approach shoes *and* a new crash pad. That's 420 bad decisions at the coffee shop.

More importantly: I'm not SSH-ing into servers at 2am because MySQL is eating all the RAM again.

## Things I'd Do Differently

Not much, honestly.

Maybe I should've set up Hugo on a subdomain first (`new.blog.trenigma.dev`), tested everything, *then* switched DNS. Would've avoided the "shit its broken" dance.

And yeah, I could've been more methodical about migrating all the old posts. But did I need every single post from 2023? No. The ones that mattered? They're in Apple Notes. The rest? Let them rest in peace in that AMI snapshot.

The images thing... yeah, I should probably make the blog prettier at some point. Add hero images. Make it look less like a 2005 tech blog. But honestly? The words are there. The site loads in 100ms. People can read it. Ship it and iterate later.

## What I Learned

**1. Static sites don't die**

There's no database to corrupt. No services to crash. No containers to run out of memory. It's just files. Files are reliable.

**2. GitHub Pages is stupid good**

Free hosting. Free SSL. Built-in CDN. `git push` deploys. Hard to beat.

**3. Backups are only useful if you can access them**

My fancy S3 database backups were useless when the instance was too broken to SSH into. You know what worked? Apple Notes syncing to iCloud.

**4. MVP > perfection**

I could've spent weeks making the Hugo blog pixel-perfect. Instead I shipped it in an afternoon with no images and minimal styling. It's fine. Nobody cares. The words matter more than the layout.

**5. Cost optimization feels good**

Saving $400/year is nice. But the real win? Not worrying about that goddamn t2 instance.

## What's Next

Now that the blog is static and costs nothing, I can focus on actually writing instead of debugging Docker networking.

I'll probably:
- Add some images eventually (when I feel like it)
- Set up monitoring (Grafana Faro RUM - that's a whole other post)
- Maybe automate image optimization in the build pipeline
- Figure out comments (GitHub Discussions? Mastodon? TBD)

But the important part? **The blog exists. It's fast. It costs nothing. And I can deploy by typing `git push`.** 🍻

---

**Current stack:**
- Hugo (static site generator)
- GitHub Pages (hosting)
- GitHub Actions (CI/CD)
- Namecheap (DNS)
- Apple Notes (apparently also my backup strategy)

**Monthly cost:** $0  
**Infrastructure headache:** Also $0  
**Time spent SSH-ing into servers at 2am:** 0 hours

If you're running a blog on EC2 or a VPS, ask yourself: do you really need a database? Or do you just need Markdown files in Git?

Because I promise you, your wallet will thank you. Your sleep schedule will thank you. And you'll spend more time writing and less time SSH-ing into servers at 2am because MySQL is eating all your RAM again.

Now go kill your AWS bill and buy some climbing gear instead.