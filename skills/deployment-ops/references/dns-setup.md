# DNS Setup Guide

## Step 1: Add Domain in Vercel

1. Open your Vercel project dashboard
2. Go to **Settings** then **Domains**
3. Enter your domain (e.g., `myapp.com`) and click **Add**
4. Vercel will show you the DNS records you need to create

## Required DNS Records

| Type | Name/Host | Value/Points to |
|------|-----------|----------------|
| A | @ (or blank) | 76.76.21.21 |
| CNAME | www | cname.vercel-dns.com |

## GoDaddy Setup

1. Log in to GoDaddy and go to **My Products**
2. Find your domain and click **DNS** (or **Manage DNS**)
3. Under **DNS Records**, click **Add New Record**

### Add the A Record
- **Type:** A
- **Name:** @ (GoDaddy uses @ for root domain)
- **Value:** 76.76.21.21
- **TTL:** 600 seconds

### Add the CNAME Record
- **Type:** CNAME
- **Name:** www
- **Value:** cname.vercel-dns.com
- **TTL:** 600 seconds

### Remove Conflicting Records
- Delete any existing A records pointing to a different IP for the root domain
- Delete any existing CNAME records for `www` that point elsewhere
- GoDaddy sometimes adds "parking" records — remove those

4. Save all changes
5. Wait 5-30 minutes for DNS to propagate
6. Go back to Vercel and verify the domain shows a green checkmark

## Cloudflare Setup

Cloudflare works differently because it acts as a proxy. You have two options:

### Option A: DNS Only (Recommended for Vercel)

1. Log in to Cloudflare and select your domain
2. Go to **DNS** then **Records**
3. Add records:

**A Record:**
- **Type:** A
- **Name:** @ (Cloudflare uses @ for root)
- **Proxy status:** DNS only (gray cloud — click the orange cloud to turn it off)
- **IPv4 address:** 76.76.21.21
- **TTL:** Auto

**CNAME Record:**
- **Type:** CNAME
- **Name:** www
- **Proxy status:** DNS only (gray cloud)
- **Target:** cname.vercel-dns.com
- **TTL:** Auto

**Important:** The proxy must be OFF (gray cloud) for Vercel's SSL to work. If the orange cloud is on, you will get SSL errors.

### Option B: Using Cloudflare Proxy (orange cloud)

If you want Cloudflare's CDN and DDoS protection in front of Vercel:

1. Set records the same as above but leave proxy ON (orange cloud)
2. In Cloudflare **SSL/TLS** settings, set encryption mode to **Full (strict)**
3. In Cloudflare **SSL/TLS** then **Edge Certificates**, enable **Always Use HTTPS**
4. This adds a layer of caching and protection but can complicate debugging

## Namecheap Setup

1. Log in to Namecheap and go to **Domain List**
2. Click **Manage** next to your domain
3. Go to **Advanced DNS** tab
4. Under **Host Records**, add:

**A Record:**
- **Type:** A Record
- **Host:** @
- **Value:** 76.76.21.21
- **TTL:** Automatic

**CNAME Record:**
- **Type:** CNAME Record
- **Host:** www
- **Value:** cname.vercel-dns.com
- **TTL:** Automatic

5. Remove any conflicting records (parking pages, default records)
6. If using Namecheap DNS, make sure the **Nameservers** section says "Namecheap BasicDNS"

## Verifying DNS Propagation

After adding records, check propagation:

1. In Vercel, go to **Settings** then **Domains** — look for green checkmarks
2. Use the terminal to check:

```bash
# Check A record
dig myapp.com A +short
# Should return: 76.76.21.21

# Check CNAME
dig www.myapp.com CNAME +short
# Should return: cname.vercel-dns.com.
```

3. If records are not showing up after 30 minutes, double-check:
   - No conflicting records exist
   - The correct DNS provider is active (check nameservers)
   - TTL on old records has expired

## Subdomain Setup

For subdomains like `app.myapp.com` or `api.myapp.com`:

1. In Vercel, add the subdomain (e.g., `app.myapp.com`)
2. At your DNS provider, add a CNAME record:
   - **Type:** CNAME
   - **Name:** app
   - **Value:** cname.vercel-dns.com

## SSL Certificates

Vercel automatically provisions and renews SSL certificates for all domains. No action needed. If SSL is not working:

1. Check that DNS records point to Vercel (not another host)
2. If using Cloudflare proxy, set SSL mode to **Full (strict)**
3. Wait up to 24 hours for certificate provisioning on new domains
