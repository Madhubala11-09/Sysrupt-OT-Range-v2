# Challenge 9 — Hidden in Plain Sight

**Protocol:** HTTP (OSINT/recon style) · **Port:** 80/tcp

---

> **Note:** like OPC Scout and Rogue Radio, I don't have confirmed data for how the login on
> `/control` or `/hist-config` is actually solved — only that `robots.txt` discloses the paths
> exist. I'm not going to invent a username/password here, since a wrong guess presented
> confidently would be worse than no guide at all. The steps below are the legitimate recon
> process up to the login form; fill in Step 4 once you know what the actual credential source is
> (another challenge's leaked value, a default-creds pattern, an engineering note, etc.) and I'll
> finish the write-up for real.

## The Story

`robots.txt` is meant to tell search engines what *not* to crawl — which means it's also a
public list of paths the site owner didn't want indexed, handed to you for free.

## What You Know

```
User-agent: *
Disallow: /control
Disallow: /hist-config
```

- `Disallow` entries are not access control — they're a polite request to crawlers, enforced by
  nothing. Anything listed there is just as reachable as any other page.
- Two paths are disclosed: `/control` and `/hist-config`.

---

## Step 1: Check robots.txt yourself

<details>
<summary>Hint</summary>

Always check this file first on any web recon — it's free information with zero risk.
</details>

<details>
<summary>Show command</summary>

```bash
curl http://10.0.1.1/robots.txt
```
</details>

## Step 2: Visit the disclosed paths directly

<details>
<summary>Hint</summary>

A `Disallow` entry is a map, not a lock. Just request the page.
</details>

<details>
<summary>Show command</summary>

```bash
curl -i http://10.0.1.1/control
curl -i http://10.0.1.1/hist-config
```

Note what each returns — a login form, a redirect, a 403, or something else entirely.
</details>

## Step 3: Look at what the login page gives away

<details>
<summary>Hint</summary>

Before trying to guess credentials, read the page source, any comments, linked JS/CSS filenames,
and response headers. CTF pages often leak more in the markup than in the visible form.
</details>

<details>
<summary>Show command</summary>

```bash
curl -s http://10.0.1.1/control | less
curl -s http://10.0.1.1/control | grep -i -E "user|pass|admin|TODO|note"
```
</details>

## Step 4: Figure out the actual credentials

<details>
<summary>Hint</summary>

*(placeholder — needs real info)*

Credentials in a range like this usually come from one of: something leaked in another
challenge (an engineering note, an SNMP community string, a historian record), a default
vendor/product credential pair disclosed in product documentation, or something embedded in the
page source itself from Step 3. Which of these applies here isn't confirmed yet.
</details>

<details>
<summary>Show command</summary>

*(fill in once the credential source is known — I don't want to guess a username/password and
present it as real)*
</details>

## What You Learned

- `robots.txt` as a reconnaissance source, not an access-control mechanism.
- Checking page source/headers for leaked hints before attempting credential guesses.
- *(to complete once Step 4 is filled in)*
