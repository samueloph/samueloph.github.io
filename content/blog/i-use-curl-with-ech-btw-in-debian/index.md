+++
title = "I use curl with ECH btw (in Debian)"
date = 2026-03-27
draft = false

[taxonomies]
categories = ["debian", "curl", "security"]

[extra]
toc = true
+++

#### tl;dr

This is an experimental feature that, for the first time, brings full ECH
support to curl on Debian using OpenSSL.

Starting with **curl 8.19.0-3+exp2** (Debian Experimental), you can now use
ECH, with HTTPS-RR and DoH for maximum privacy.

curl 8.19.0-3+exp2 is quite fresh at the time of writing, bear in mind that your
repository might not have synced the package yet, all mirrors should have it by
March 27th 14:00 UTC.

```bash
# defo.ie is a test server that confirms whether ECH was successfully used
curl -v --ech hard https://defo.ie/ech-check.php
# For Encrypted Client Hello (ECH) + DNS over HTTPS (DoH)
curl -v --ech hard --doh-url https://1.1.1.1/dns-query https://defo.ie/ech-check.php
```
*"--ech hard" tells curl to refuse the connection entirely if ECH cannot be negotiated.*

Or, if you would like to try it out in a container:
```bash
podman run debian:experimental /bin/bash -c 'apt install --update -t experimental -y curl && curl -v --ech hard --doh-url https://1.1.1.1/dns-query https://defo.ie/ech-check.php'
```
*(in case you haven't noticed, apt now has the `--update` option for the
`upgrade` and `install` commands)*

# For Privacy
CloudFlare calls it "the last puzzle piece to privacy" in their must-read
announcement: [https://blog.cloudflare.com/announcing-encrypted-client-hello/](https://blog.cloudflare.com/announcing-encrypted-client-hello/).

[Encrypted Client Hello (rfc9849)](https://www.rfc-editor.org/rfc/rfc9849) encrypts the
"which website are you connecting to?" part of the TLS handshake that was
previously visible in plaintext.

[HTTPS-RR (rfc9460)](https://www.rfc-editor.org/rfc/rfc9460) is a DNS record type that
publishes connection parameters for a service, including the public key clients
need to perform ECH.

[DNS Over HTTPS (rfc8484)](https://www.rfc-editor.org/rfc/rfc8484) encrypts DNS queries
by tunneling them over HTTPS, hiding what domains you're looking up from
network observers.

When all three operate together over a CDN with shared IP space, the target
domain name is hidden from passive observers; the HTTPS-RR record is queried
over DoH in order to [retrieve the ECH key
(rfc9848)](https://www.rfc-editor.org/rfc/rfc9848) for the TLS handshake.

Seems like quite an important feature, and in fact the major browsers have it
enabled for some time now, the trick is that they do not use OpenSSL (Chrome
uses BoringSSL and Firefox uses NSS).

For everyone else, the only option is to patch OpenSSL or wait until 4.0.0 is
released, and so part of the reason Debian is the first distro to enable it
(curl + OpenSSL + ECH) is that the OpenSSL maintainer (Sebastian Andrzej
Siewior) packaged the alpha release just 3 days after it was published.

Do not forget that ECH support is experimental and currently relies on the
alpha release of OpenSSL.

# wcurl Gets It Too

Considering [wcurl](https://curl.se/wcurl/) is just a wrapper on curl, it gets
the feature for free:

```bash
wcurl --curl-options="--ech hard --doh-url https://1.1.1.1/dns-query" $URL
```

If you're using wcurl, you don't want to have to set parameters, this is just
to show that the feature is there and if you have a `.curlrc` file, it can
enable the feature seamlessly.

# Other Debian Releases

Given the ECH feature requires OpenSSL >= 4, it will not make it to Debian 13,
having a small chance of going to Debian 13 Backports (emphasis on small).

**It should get to Debian Unstable and Debian Testing within the next couple of
months** as the OpenSSL GA release happens and gets packaged, but you should be
able to install the package from Experimental in your Unstable and Testing
systems without issues. It will also be in Debian 14 once it becomes the new Stable.

# Shoulders of Giants

Stephen Farrell's presentation from OpenSSL Conference 2025 has a lot of
background on the work involved:

[Encrypted Client Hello – Lessons learned from trying to do something that was
probably too complicated](https://www.youtube.com/watch?v=wYQq8ozP3uE)

They have been working on implementing ECH in open-source projects for years,
something as big as this doesn't happen without lots of people dedicating both
their paid and free times over it.

I ended up being the person who enabled it on Debian, which was pretty much the
least amount of work between everyone involved, but hey it's fun flipping the
switch and telling you about it.

# Background

Since 2025, the curl developers started organizing an yearly meeting with all
maintainers of curl in Operating Systems. The 2026 edition happened in March
26th:
[https://github.com/curl/curl/wiki/curl-distro-discussion-2026](https://github.com/curl/curl/wiki/curl-distro-discussion-2026).

Attendance was really good, and as you can imagine one of the topics of
discussion was ECH, in which it was pointed out that having OpenSSL 4 was
the main requirement but besides it nothing unusual was needed.

In Debian Experimental, we have been enabling HTTPS-RR since March 2025, and
OpenSSL 4.0.0 alpha was packaged just recently (2026-03-13) by Sebastian
Andrzej Siewior, it's time for the next step.

The curl distro meeting was just the motivation I needed to go ahead and
enable it in Debian Experimental, so as part of our Debian Brasil Weekly
Meetings I've prepared and uploaded the changes, while Carlos Henrique Lima
Melara worked on addressing a recent test regression for Debian Unstable.
Unfortunately sergiodj couldn't join and I'm sure he's jealous of the hacking
session now.

# Appendix
While writing this, I've noticed one of the authors of the CloudFlare blogpost
is the previous curl maintainer on Debian; Alessandro Ghedini let me take over
the maintenance back in 2021 and today curl is maintained by a team of 4
people, it's nice to see Alessandro's involvement.
