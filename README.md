# residential socks5 proxy: how to choose a plan, wire it into anti-detect browsers or scripts, and pay per IP instead of per gigabyte

Most people searching this term already know what they want. They need IPs that belong to real home connections, they need the SOCKS5 handshake rather than an HTTP proxy, and they want two answers: what does this cost, and how do I plug it in. The part that trips people up is that providers price residential access in two completely different units, and the unit decides whether your bill lands at $24 or $180 for the same workload.

Below is the practical version — what SOCKS5 changes, which jobs fit it, how per-IP and per-GB billing diverge, and how the credentials arrive from a provider like 9Proxy.

## What you're actually renting

A residential IP is an address handed out by an ISP to a real device — a home router, a phone on mobile data, a consumer laptop. To a target server, traffic arriving from it looks like a normal person in a normal house. That's the entire point of paying a premium over datacenter proxies, which sit in known server subnets and get spotted at the IP reputation layer before your request even reaches the application.

SOCKS5 is the other half. It's a transport-layer protocol: it forwards TCP connections (and UDP, if the provider supports it), passes your traffic through untouched, and doesn't interpret HTTP at all. Three consequences matter in practice:

- It carries traffic that isn't HTTP/HTTPS, so non-web protocols and tools like proxychains work without conversion.
- It can hand DNS resolution to the proxy. With `socks5://` your machine resolves the hostname first; with `socks5h://` the hostname goes to the proxy and it resolves there. For geo-targeting that difference is not cosmetic.
- It doesn't rewrite headers, which is why anti-detect browsers and custom scripts tend to prefer it — nothing in the request gets silently altered.

There's a catch worth knowing before you buy anything. Chrome doesn't support SOCKS5 proxy authentication, so a username-and-password SOCKS5 endpoint won't work in a normal Chrome session. Use Firefox, use an anti-detect browser, or authenticate by IP whitelist. Some target sites also flag SOCKS5-shaped traffic outright; if you get connection errors that make no sense, testing the same endpoint over HTTP/HTTPS is the fastest way to find out.

> Protocol choice is not a loyalty question. If a site blocks or breaks your SOCKS5 session, switching the same credentials to HTTP/HTTPS costs you one dropdown change.

## SOCKS5 vs HTTP proxy: pick by what the job needs

|  | SOCKS5 | HTTP/HTTPS proxy |
| --- | --- | --- |
| Traffic it carries | TCP, plus UDP where supported | TCP only (HTTP CONNECT) |
| Header handling | Passes request through | Can add or rewrite headers |
| Auth in Chrome | Not supported | Supported |
| Good fit | Anti-detect browsers, proxychains, Python/Node scripts, non-HTTP traffic | Browser sessions, tools built around header-based targeting |
| Targeting style | Often encoded in the username string | Usually in the username string too, sometimes in the entry point |

If your tooling already supports SOCKS5, use SOCKS5. If your tooling is a plain browser tab, HTTP is less friction.

## The billing decision that actually sets your price

Residential providers charge either by the number of IPs or by the gigabytes consumed. These are not interchangeable, and 9Proxy sells both, which makes the trade-off easy to see.

Per-IP plans give you a fixed number of addresses with unlimited bandwidth. In 9Proxy's case, unused IPs don't expire, and each IP stays live for a few hours up to about 24 hours. That model suits anything session-based: account work, long-running scrapes, anything where a mid-task IP change breaks the job.

Per-GB plans flip it. There's no fixed IP allocation — you generate as many endpoints as you want and only the traffic is metered, with rotation on every request or sticky sessions you control. Traffic validity runs 180 days, and the enterprise tiers don't expire at all.

Run the numbers on your own job before picking:

- 2 GB of traffic on the 5 GB pack costs $6. One IP with unlimited bandwidth costs $0.24.
- 15 GB across 40,000 requests costs $22.50 at the $1.50/GB tier — cheap if each request is light and you're rotating across countries, expensive if a single sticky IP with unlimited bandwidth would have done the job.

The rule of thumb that holds up: bandwidth-heavy but session-light workloads belong on per-GB. Session-heavy workloads belong on per-IP. Rotating across thousands of cities belongs on per-GB, because per-IP plans bill 1 IP = 1 usage when forwarded, and burning an IP per request gets wasteful fast.

## How 9Proxy hands you a residential SOCKS5 endpoint

9Proxy runs a residential pool of 20M+ IPs across 90+ countries, with targeting down to country, state, city, ZIP code and ISP, and supports HTTP/HTTPS alongside SOCKS5. Two product models, two different setup paths:

**Per-IP access.** You run the 9Proxy desktop app, filter the pool by location, right-click an IP, and forward it to a local port. The app handles the connection; you paste `host:port` into your tool and select SOCKS5. Optional proxy authentication can be switched on. Third-party integration guides for ixBrowser, MuLogin, VMLogin, XLogin and ClonBrowser all follow that same pattern, because it's how the IP-based product works by design.

**Per-GB access.** No desktop app needed. You create a sub-account in the dashboard, generate a proxy session, and get a host, port, username and password that you can use directly in an anti-detect browser profile or a script. Authentication is username/password or IP whitelist.

The username isn't random — it's a targeting string:


<subaccount>-country-<country_code>-st-<state_code>-city-<city_code>-isp-<isp_code>-ssid-<session_id>-sst-<session_time>


A minimal example looks like `useruser123-country-US-ssid-rhdN1907ma`, with your sub-account password as the password. Country, city, ISP, session ID and session lifetime all ride inside that string, which is why you can rotate or pin without touching your code.

👉 [Create a 9Proxy account and generate your first residential SOCKS5 session](https://bit.ly/9-Proxy)

A few operational details worth knowing: the provider advertises 99.95% uptime and a "Today List" feature that lets you reuse IPs issued in the last 24 hours, which cuts spend if your targets don't need fresh addresses every run. Support is 24/7 human, reachable by Telegram, email and ticket — Geekflare's review specifically called out the absence of a chatbot loop as a plus. That review also ran 300 sequential requests against a Cloudflare-protected e-commerce platform and reported 293 passes (97.7%), 5 CAPTCHAs and 2 hard blocks, with a 0.63-second average response time.

## Setting it up: browser, cURL, Python

**In an anti-detect browser (per-GB endpoint):**

1. In your 9Proxy dashboard, create a sub-account and generate a proxy session.
2. Open your browser tool — AdsPower, ixBrowser, Incogniton and similar all follow the same layout.
3. Create a new profile and go to proxy settings.
4. Set proxy type to SOCKS5.
5. Enter the host and port from the session you generated.
6. Paste the structured username and your sub-account password.
7. Run the built-in proxy check. You should see the exit IP and its city.

**In a terminal:**

bash
curl --socks5-hostname USERNAME:PASSWORD@HOST:PORT https://api.ipify.org


`--socks5-hostname` pushes DNS to the proxy. Plain `--socks5` resolves locally, which can leak your real resolver location into the request path.

**In Python:**

bash
pip install "requests[socks]"


python
import requests

proxy = "socks5h://USERNAME:PASSWORD@HOST:PORT"
r = requests.get("https://api.ipify.org", proxies={"http": proxy, "https": proxy}, timeout=30)
print(r.text)


Swap `HOST`, `PORT`, `USERNAME` and `PASSWORD` for the values in your session. If the response is your expected city rather than your own ISP, the endpoint is working.

## What residential SOCKS5 actually costs

IP-based plans, unlimited bandwidth per IP, and unused IPs don't expire:

| Plan | IPs delivered | Price per IP | Total | Buy |
| --- | --- | --- | --- | --- |
| 100 IPs | 100 | $0.24 | $24 | [ Get the 100 IP plan](https://bit.ly/9-Proxy) |
| 500 IPs | 500 | $0.144 | $72 | [ Get the 500 IP plan](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | 1,500 | $0.084 | $126 | [ Get the 1,000 IP plan](https://bit.ly/9-Proxy) |
| 2,500 IPs | 2,500 | $0.084 | $210 | [ Get the 2,500 IP plan](https://bit.ly/9-Proxy) |
| 5,000 IPs | 5,000 | $0.072 | $360 | [ Get the 5,000 IP plan](https://bit.ly/9-Proxy) |
| 15,000 IPs | 15,000 | $0.048 | $720 | [ Get the 15,000 IP plan](https://bit.ly/9-Proxy) |
| 100,000 IPs | 100,000 | $0.023 | $2,300 | [ Get the 100,000 IP plan](https://bit.ly/9-Proxy) |
| 500,000 IPs | 500,000 | ≈$0.017 | $8,625 | [ Get the 500,000 IP plan](https://bit.ly/9-Proxy) |

Intermediate volume tiers exist between 15,000 and 100,000 IPs (roughly $0.035/IP at 25,000 and $0.029/IP at 50,000), so confirm the exact figure in the dashboard before committing to bulk.

GB-based plans, billed by traffic with rotation included:

| Plan | Price per GB | Total | Traffic validity | Buy |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [ Get the 5 GB plan](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [ Get the 50 GB plan](https://bit.ly/9-Proxy) |
| 100 GB | $1.50 | $150 | 180 days | [ Get the 100 GB plan](https://bit.ly/9-Proxy) |
| 200 GB | $1.00 | $200 | 180 days | [ Get the 200 GB plan](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80 | $800 | 180 days | [ Get the 1,000 GB plan](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [ Get the 2,000 GB plan](https://bit.ly/9-Proxy) |
| 3,000 GB (Enterprise) | $0.72 | $2,160 | No expiry | [ Get the 3,000 GB plan](https://bit.ly/9-Proxy) |
| 6,000 GB (Enterprise) | $0.70 | $4,200 | No expiry | [ Get the 6,000 GB plan](https://bit.ly/9-Proxy) |
| 10,000 GB (Enterprise) | $0.68 | $6,800 | No expiry | [ Get the 10,000 GB plan](https://bit.ly/9-Proxy) |

Bundle plans, if you want both IPs and traffic in one purchase:

| Bundle | Contents | Price | Buy |
| --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | [ Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | [ Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | [ Get the Pro bundle](https://bit.ly/9-Proxy) |

Two pricing notes that matter for timing. On June 1, 2026, 9Proxy adjusted prices for IP-based and bundle packages — the current floor sits around $0.018/IP. GB-based pricing was explicitly left unchanged, which is why the GB table still starts at $3.00/GB and bottoms out at $0.68/GB. Promo codes rotate constantly (past examples include a seasonal 8% code and a 9% GB coupon), so treat any code you see quoted online as expired until the checkout page accepts it. Signing up through the invite link applies the new-user referral discount their partner program lists at 5% for referred users.

👉 [Sign up and check the plan that fits your workload](https://bit.ly/9-Proxy)

## Check these five things before you pay

1. **Whether you can test first.** There's no self-serve free trial button on the site. Trials are granted on request and depend on availability — the provider runs trial requests through its community threads, so budget a day of back-and-forth before assuming you'll evaluate for free.
2. **Your billing model.** If your job keeps one session alive for hours, per-GB traffic charges will feel arbitrary. If you rotate across cities per request, per-IP blocks will empty out faster than the sticker price suggests.
3. **Refund expectations.** Geekflare notes that the bulk of 9Proxy's Trustpilot friction comes from refund policy complaints rather than the proxy network itself, and users who bought a mismatched plan had trouble recovering the spend. That's an argument for a small first purchase, not a large one.
4. **SOCKS5 authentication path.** Confirm up front whether you'll authenticate with user/pass or an IP whitelist. If your setup is standard Chrome, neither a password-protected SOCKS5 endpoint nor (often) a port-forwarded one will work there — plan around a different browser or a tool that supports SOCKS5 natively.
5. **Interfaces you actually use.** AdsPower, ixBrowser, Incogniton, MuLogin, VMLogin, XLogin, Hidemium and ClonBrowser all have documented 9Proxy setup flows. If your tool isn't on that list, confirm it accepts `host:port:user:pass` SOCKS5 before buying.

## FAQ

**Is SOCKS5 better than HTTP for residential proxies?**
Not universally. SOCKS5 moves arbitrary TCP and UDP traffic without touching headers, which is why anti-detect browsers and scripts prefer it. HTTP/HTTPS is easier for browser-only work and is the only one of the two that Chrome will authenticate against.

**Why do I get connection errors with a residential SOCKS5 proxy?**
Two common causes: DNS resolution happening locally (use `socks5h://` or `--socks5-hostname`) and the target site actively flagging SOCKS5 traffic. The second one is a one-dropdown fix — switch the same endpoint to HTTP/HTTPS and retest.

**How long does a residential IP last on a per-IP plan?**
A few hours up to around 24 hours, varying per address. Per-IP plans don't rotate on their own, so if you need rotation you either enable the auto-rotation option on selected ports or move to a per-GB plan with rotating or sticky modes.

**What happens to unused traffic or IPs?**
IPs on per-IP plans don't expire. Traffic on the standard GB packages is valid for 180 days, while the enterprise GB tiers carry no expiry at all — that's the main reason teams with uneven project volume drift toward the larger GB packs.

## Where this lands

If your work is session-based and bandwidth-hungry, the per-IP side of 9Proxy is hard to beat on price — $0.24 per IP with unlimited transfer is a different order of magnitude from per-gigabyte metering once a single IP is pulling tens of gigabytes. If your work rotates constantly across regions and each request is small, the GB packs at $1.50–$0.80 per GB do the job with less waste.

Pick the unit before you pick the vendor. That decision has more effect on your monthly spend than any promo code.

👉 [Start with 9Proxy and generate your first SOCKS5 credentials in a few minutes](https://bit.ly/9-Proxy)
