---
layout: post
title: "Go Pro for Free: Secure Your GitHub Pages Blog with HTTPS & a Custom Domain"
description: "Unlock the secrets to launching a professional, free blog using GitHub Pages on your custom domain. This post provides a comprehensive guide to hosting your blog on an apex domain and subdomain, guaranteeing advanced security and performance without breaking the bank."
date: 2026-04-12 11:38:11 -0400
categories: [Blog]
tags: [github-pages, dns, security, custom-domain]
---

GitHub Pages is one of the best free hosting options available, but the default `<YOUR_USERNAME>.github.io` URL does nothing for your credibility. Pair it with a custom domain that has been HSTS preloaded and you get a professional, fully secured site at near-zero cost. This guide walks through the whole chain: wiring up DNS, getting HTTPS provisioned, and validating the site is working properly.

## Prerequisites

- A GitHub repository with GitHub Pages enabled (source branch configured)
- A registered custom domain (see Step 1 below)
- Access to your domain registrar's DNS management panel

## Step 1: Register a Custom Domain

Any major registrar works - Namecheap, Cloudflare, and SquareSpace are some of the options I have used in the past. Cloudflare Registrar is worth considering because it sells domains at cost (no markup) and its DNS management UI is excellent. I picked a `.dev` domain so it is HSTS preloaded by Google already, so I don't have to worry about setting that up.

Once you have a domain, keep the registrar's DNS management panel open - you'll need it shortly.

## Step 2: Configure DNS Records

GitHub Pages requires specific DNS records depending on whether you want to use your apex domain (`kylefender.dev`), a `www` subdomain, or both.

### Apex domain

In your DNS management panel, add four A records pointing to GitHub's servers. I did this using CloudFlare - make sure not to proxy the requests.

| Type | Name | Value             |
| ---- | ---- | ----------------- |
| A    | `@`  | `185.199.108.153` |
| A    | `@`  | `185.199.109.153` |
| A    | `@`  | `185.199.110.153` |
| A    | `@`  | `185.199.111.153` |

### `www` subdomain

In your DNS management panel, add a CNAME record. I did this using CloudFlare - make sure not to proxy the requests.

| Type  | Name  | Value                              |
| ----- | ----- | ---------------------------------- |
| CNAME | `www` | `<your-github-username>.github.io` |

Adding records for both the apex and subdomain, lets GitHub redirect `www` to your apex domain (or vice versa) automatically. This isn't readily apparent until you curl the location during the verification steps at the end; it is redirecting.

> DNS propagation can take anywhere from a few minutes to 48 hours. You can check the status with `dig kylefender.dev +noall +answer` or a tool like [dnschecker.org](https://dnschecker.org).
{: .prompt-info}

## Step 3: Add the Custom Domain in GitHub Pages Settings

1. Navigate to your blog repository on GitHub.
2. Click **Settings** in the top menu, then **Pages** in the left sidebar.

    > Regarding the next step. In the Build and deployment section of the Pages view, if the **Source** is **GitHub Actions**, nothing additional will happen. If the **Source** is **Deploy from a branch**, GitHub will commit a `CNAME` file to the root of your pages source branch and begin a DNS check.
    {: .prompt-info }

3. Under **Custom domain**, enter your domain (ex. `www.kylefender.dev`) and click **Save**.

    > If you are going to host the blog at the apex domain, and `www` subdomain, the **Custom domain** you enter must include `www`, or else you will get certificate errors when navigating to the `www` subdomain.
    {: .prompt-tip }

The GitHub documentation says to wait for the DNS check to pass before moving on, and that the **Pages** settings page will show a green checkmark once it verifies. I didn't see a green checkmark, and everything worked, so I don't know what happened.

## Step 4: Provision HTTPS (Let's Encrypt)

Once the DNS check passes, GitHub Pages automatically requests a TLS certificate from Let's Encrypt for your domain. This usually completes within a few minutes.

1. Stay on the **Settings → Pages** page.
2. Once the certificate is ready, the **Enforce HTTPS** checkbox becomes available.
3. Check **Enforce HTTPS**.

This does two things: it redirects all `http://` traffic to `https://`, and it sets the `Strict-Transport-Security` response header on every request. You're already protected from downgrade attacks at this point.

## Step 5: Verifying Everything Works

Run a quick end-to-end check:

```bash
# Confirm HTTPS redirect
curl -sI http://kylefender.dev | grep -i location

# Confirm www redirects to apex (or vice versa) - you shouldn't see anything here because the apex is being redirected ot the subdomain
curl -sI https://www.kylefender.dev | grep -i location
```
{: .nolineno }

You can also run your domain through [SSL Labs](https://www.ssllabs.com/ssltest/) for a full certificate and security header report.

## Wrapping Up

With a custom domain, GitHub-provisioned Let's Encrypt certificate, and HTTPS enforcement in place, your GitHub Pages site is as secure as any paid hosting setup. The only cost is the domain registration itself - typically $10–15/year.
