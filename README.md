# playwright rotating proxy: rotating IPs per browser context without 407s, dropped sessions, or burned GB

You wrote four lines of Playwright, passed a proxy URL, and got a **407 Proxy Authentication Required** back. Nothing in the traceback points at the proxy. The page just doesn't load.

That's the first wall almost everyone hits, and it isn't a proxy provider problem. It's how Playwright reads proxy config. Get past it and you hit the second wall, which is more expensive: the rotation strategy you copied rotates the wrong thing, and the IPs change while the browser fingerprint stays identical.

This walks through both, with working code, and uses DataImpulse's residential gateway as the exit layer so the numbers in the examples are real.

## Playwright throws away credentials in the proxy URL

Both `requests` and Scrapy accept `http://user:pass@host:port` as-is. Playwright doesn't. If you hand it that single string, the username and password are dropped silently, and the failure surfaces later as a 407 or a connection that never completes.

The proxy option wants the parts split out:

python
from playwright.sync_api import sync_playwright

proxy = {
    "server": "http://gw.dataimpulse.com:823",
    "username": "YOUR_LOGIN",
    "password": "YOUR_PASSWORD",
}

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context(proxy=proxy)
    page = context.new_page()
    page.goto("https://httpbin.org/ip")
    print(page.inner_text("body"))
    context.close()
    browser.close()


`page.goto("https://httpbin.org/ip")` returning a residential IP instead of your server's address is the whole test. Two lines, no guessing.

## You cannot change the proxy mid-context

This is the constraint that decides your rotation design. `proxy` is set when the browser launches or when a context is created. There is no API to swap it inside a live context.

So rotation in Playwright always means the same thing: **create a new context**. Either pattern below works.

**Fresh IP per connection.** Point the browser at a rotating gateway with no session identifier in the credentials. Every new connection to the target picks a new exit IP. You create one context per batch and let the gateway handle churn underneath.

python
context = browser.new_context(
    proxy={
        "server": "http://gw.dataimpulse.com:823",
        "username": "YOUR_LOGIN__cr.us",
        "password": "YOUR_PASSWORD",
    },
    locale="en-US",
    timezone_id="America/New_York",
)


**One IP per session.** Add a session ID to the username so the gateway pins one exit IP for the life of that identifier. DataImpulse holds sticky sessions for around 30 minutes.

python
def sticky_proxy(session_id: str, country: str = "us") -> dict:
    return {
        "server": "http://gw.dataimpulse.com:823",
        "username": f"YOUR_LOGIN__cr.{country};sessid.{session_id}",
        "password": "YOUR_PASSWORD",
    }


Country targeting rides in the username the same way (`__cr.de`, `__cr.gb`). The exact string varies slightly by account setup, so copy the one the dashboard generates for you rather than reconstructing it from a blog post — mine included.

## Rotating per request is usually the wrong move

There's a common pattern in tutorials: pull a random proxy from a list for every `page.goto()`, or rotate inside a request interceptor. Both fight the way a browser session actually looks.

If you swap the exit IP halfway through a flow, you've created a visitor whose cookies say they logged in from Germany and whose next request arrives from Japan. That's not two anonymous requests. It's one identity contradicting itself, and it's easier to detect than a single mediocre IP. Session-aware systems score exactly this.

The unit to rotate is the thing that corresponds to a visitor:

| Rotation unit | Isolates | Best for |
| --- | --- | --- |
| Per request | Nothing meaningful | Stateless API polling, no cookies, no login |
| Per context | Cookies, storage, cache, exit IP | Multi-account work, parallel independent jobs |
| Per browser process | Storage plus fingerprint material | Targets that compare sessions against each other |

One context, one proxy, for the life of that task. Close it, move the next job to a different exit. If your target only counts requests per address, per-context is enough and it's cheaper. If it fingerprints and cross-references sessions, per-context isn't enough — see the next section.

## Per-context proxies rotate the IP, not the machine

Here's the part most guides skip. A `BrowserContext` isolates storage. It does not isolate hardware. Every context in one browser process reports the same canvas hash, the same WebGL renderer, the same font list, the same audio profile, the same screen dimensions, and the same TLS handshake shape.

Five contexts on five residential IPs are not five users. They're one machine appearing from five countries simultaneously, and the constant fingerprint links them to each other.

What a context genuinely separates: cookies and jars, local storage, IndexedDB, cache and service workers, permissions, and the proxy when set per context. That's real and useful. If your job is "log into ten accounts without them seeing each other's cookies," contexts are the correct tool.

What it shares: everything the machine reports about itself. Anti-bot systems score that before your page finishes loading.

If a target compares sessions to each other, one process per identity is the pattern that works, not contexts in one process. The trade is straightforward — more memory, more launch overhead, better separation.

## Match timezone, locale, and language to the exit IP

This one bites careful people because it looks like a setting they already configured.

A US residential exit paired with `locale: "de-DE"` is an inconsistency no real browser produces. Resolve timezone and locale per context, from the proxy's country, every time:

python
GEO = {
    "us": ("en-US", "America/New_York"),
    "de": ("de-DE", "Europe/Berlin"),
    "gb": ("en-GB", "Europe/London"),
}

def context_for(browser, country: str, session_id: str):
    locale, tz = GEO[country]
    return browser.new_context(
        proxy=sticky_proxy(session_id, country),
        locale=locale,
        timezone_id=tz,
        viewport={"width": 1920, "height": 1080},
    )


If a tool resolves timezone once at browser launch, it resolved it for the launch-time exit. Every context created afterwards carries that zone regardless of where its own proxy leaves from. Context one leaves Germany with a German timezone. Context two leaves Japan — still with a German timezone. Keep the mapping explicit and per context.

## SOCKS5 doesn't authenticate in Playwright

Most proxy pools that matter ship SOCKS5 with per-account credentials. Playwright documents username and password for HTTP proxies only.

Pass credentials on a `socks5://` server and nothing raises. It simply doesn't authenticate, and the failure shows up later as requests that silently fall through or never arrive. If you need authenticated rotation, use the HTTP/HTTPS gateway endpoint — DataImpulse serves it on `gw.dataimpulse.com:823`, with SOCKS5 on port 824 for use cases where IP whitelisting replaces credentials.

## Failure handling that actually keeps a job running

A proxy pool degrades. Treat individual failures as routing information, not as crashes.

- Blacklist a proxy on `ERR_PROXY_CONNECTION_FAILED` or repeated 407s, and don't retry the same address immediately.
- Retry with backoff on `ERR_TUNNEL_CONNECTION_FAILED`, because that's often transient.
- Rotate to a fresh session on a CAPTCHA wall rather than hammering the same address — the IP is already scored.
- Keep the session-proxy mapping stable for the whole task, so a retry inside one flow doesn't arrive from a new country.
- Watch cost per **successful** request, not cost per gigabyte. A pool at half the price with two-thirds the success rate costs more per useful page.

Placeholder point worth internalizing: retries and failed loads still burn traffic on most providers. A page that takes three attempts to load costs you three times the bytes.

## What DataImpulse is, and why it fits this setup

DataImpulse is a Cyprus-headquartered proxy provider that launched in 2022 and competes almost entirely on billing model. Residential traffic is **$1 per GB** at any volume, pay-as-you-go, with no subscription and no monthly minimum. Traffic you buy never expires.

For the Playwright case specifically, a few things matter:

- A standard `http://user:pass@host:port` endpoint that drops into Playwright's proxy option with country and session targeting carried in the username — no custom plumbing.
- Pool advertised at 90M+ residential IPs across 195 countries, HTTP/HTTPS and SOCKS5.
- Country-level targeting included in the base rate. City, state, ZIP and ASN targeting are paid add-ons, which is worth pricing before you build a location-specific pipeline around them. Third-party breakdowns disagree on the exact multiplier here — one reports 2× the standard per-GB rate, while a review site describes city targeting as bundled at no premium. Confirm with support for your specific plan before you commit a budget to it.
- Sticky sessions hold for roughly 30 minutes, which is on the shorter side if a flow runs long.
- Published success rate of 99.51%. Independent benchmarks are more sober: Proxyway's April 2025 round measured roughly 70–75% success on hard targets like Google and Instagram on standard residential. It's a mid-tier pool at a bottom-tier price, not an enterprise network.

The honest framing: this is a budget-first provider. If your targets are protected, expect to spend part of your traffic budget on retries unless you're using their per-request billing option. If your targets are moderately defended and your retry layer is sane, the arithmetic works.

The minimum entry is **$5**, which is the point — you can validate it against your own targets before deciding anything.

👉 [Start with the $5 intro pack and test it against your real targets](https://bit.ly/dataimPulse)

## All DataImpulse plans, current as published

Every product uses the same model: pay-as-you-go, traffic never expires, no subscription.

| Product | Plan | Traffic | Price | Per GB | Get it |
| --- | --- | --- | --- | --- | --- |
| Residential | Intro | 5 GB | $5 | $1.00 | [Get the 5 GB intro pack](https://bit.ly/dataimPulse) |
| Residential | Basic | 50 GB | $50 | $1.00 | [Get 50 GB residential](https://bit.ly/dataimPulse) |
| Residential | Advanced | 1 TB | $800 | $0.80 | [Get the 1 TB residential tier](https://bit.ly/dataimPulse) |
| Residential | Custom+ | 5 TB+ | From $4,000 | Custom | [Request residential volume pricing](https://bit.ly/dataimPulse) |
| Datacenter | Intro | 10 GB | $5 | $0.50 | [Get the 10 GB datacenter pack](https://bit.ly/dataimPulse) |
| Datacenter | Basic | 100 GB | $50 | $0.50 | [Get 100 GB datacenter](https://bit.ly/dataimPulse) |
| Datacenter | Advanced | 1 TB | $450 | $0.45 | [Get the 1 TB datacenter tier](https://bit.ly/dataimPulse) |
| Datacenter | Custom+ | 5 TB+ | From $2,250 | Custom | [Request datacenter volume pricing](https://bit.ly/dataimPulse) |
| Mobile | Intro | 2.5 GB | $5 | $2.00 | [Get the 2.5 GB mobile pack](https://bit.ly/dataimPulse) |
| Mobile | Basic | 25 GB | $50 | $2.00 | [Get 25 GB mobile](https://bit.ly/dataimPulse) |
| Mobile | Advanced | 1 TB | $1,600 | $1.60 | [Get the 1 TB mobile tier](https://bit.ly/dataimPulse) |
| Mobile | Custom+ | 5 TB+ | From $8,000 | Custom | [Request mobile volume pricing](https://bit.ly/dataimPulse) |
| Premium Residential | Intro | 1 GB | $5 | $5.00 | [Get the 1 GB premium pack](https://bit.ly/dataimPulse) |
| Premium Residential | Basic | 10 GB | $50 | $5.00 | [Get 10 GB premium residential](https://bit.ly/dataimPulse) |
| Premium Residential | Custom+ | 5 TB+ | From $20,000 | Custom | [Request premium residential pricing](https://bit.ly/dataimPulse) |

Rates move. Check the current figures on the pricing page before you build a cost model on this table.

## Which tier to route your Playwright job through

Don't default to residential because it's the popular option. Route each job to the cheapest tier that actually works.

**Datacenter at $0.50/GB** is right for unprotected targets, your own staging environments, public datasets, and the prototyping phase where you're debugging your rotation logic rather than collecting data. Most Playwright rotation code gets written and tested here for half the price.

**Residential at $1/GB** is the workhorse. Anything with real bot protection, e-commerce listings, SERPs, ad verification, geo-restricted content. This is the default for the per-context patterns above.

**Mobile at $2/GB** earns its price only on mobile-first platforms and app APIs, and on targets that specifically discount non-carrier traffic. Don't use it as a general upgrade.

**Premium residential at $5/GB** is 5× the standard rate for a filtered pool and an account manager. You need a specific reason — a target where the standard pool's success rate is measurably below your threshold.

One practical note on volume: buying 1 TB to reach $0.80/GB saves you $200 on the sticker price, and the traffic doesn't expire, so there's no month-end waste. That's still $800 committed before you know your success rate on your actual targets. Run the $5 pack first.

## A short test to run before you scale

You can decide most of this in an afternoon, on your own targets:

1. Buy the smallest pack.
2. Point your Playwright script at a ten-URL sample of the real target — not a demo site.
3. Count successful extractions, not requests sent.
4. Divide your spend by successes. That's your actual cost per result.
5. Re-run the same sample on datacenter exits to see whether you were paying for residential unnecessarily.

If the success rate is acceptable, scale. If it isn't, the fix is usually the rotation strategy — sticky where you were rotating, or one context per job where you were sharing — not a more expensive provider.

## Mistakes that cause most Playwright proxy problems

- Credentials left inside the proxy URL, producing silent auth failure.
- Trying to change the proxy inside a live context. Not possible — new context or restart.
- Rotating mid-flow on a login or checkout sequence. The session breaks and the account gets flagged.
- Assuming per-context rotation hides your fingerprint. It doesn't; it isolates storage only.
- Mismatched locale and exit geography. A US IP with `de-DE` locale is an obvious inconsistency.
- Using `socks5://` with credentials and no error being raised.
- Testing on `example.com`, which has no bot defence, and concluding the setup works.
- Measuring cost per GB instead of cost per successful request.

## FAQs

**Why do I get a 407 with a correct proxy URL?**
Playwright ignores credentials embedded in `proxy.server`. Pass `username` and `password` as separate keys in the proxy dict.

**Can I switch proxies without restarting the browser?**
You can't switch inside a context. Create a new `BrowserContext` with its own proxy, or relaunch. Context creation is cheap enough to make this the standard rotation pattern.

**Rotating or sticky — which one?**
Rotating for independent page fetches with no session state. Sticky for anything with a login, cart, or multi-step funnel. Most real Playwright stacks need both and switch per job type.

**Does DataImpulse have a free trial?**
No. Access starts at a $5 minimum purchase. Intro plans carry a 7-day money-back guarantee for card payments, provided less than 80% of the traffic has been consumed. Crypto purchases on intro plans are non-refundable, and follow-up top-ups carry a $50 minimum — worth knowing before you build a habit of $5 top-ups.

**Will residential proxies alone stop me getting blocked?**
No. A proxy changes your IP and nothing else. Realistic user agents, matching viewport and locale, per-session fingerprints, and sane concurrency limits all do work your proxy can't.

**How do I check which IP a context is actually using?**
`page.goto("https://httpbin.org/ip")` and read the body. Do it right after `new_context()` with the proxy applied — if it returns your own server address, the proxy was never in effect.
