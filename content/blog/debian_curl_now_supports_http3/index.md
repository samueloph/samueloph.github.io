+++
title = "Debian's curl now supports HTTP3"
date = 2024-07-04
draft = false

[taxonomies]
categories = ["debian", "curl"]

[extra]
toc = true
thumbnail = "debian_curl_http3_image.png"
+++

#### tl;dr

Starting with **curl 8.0.0-2**, you can now use HTTP3.
```bash
curl --http3-only https://example.com
```

Or, if you would like to try it out in a container:
```bash
podman run debian:unstable apt install --update -y curl && curl --http3-only https://example.com
```
*(in case you haven't noticed, apt now has the `--update` option for the
`upgrade` and `install` commands, although not available on stable yet)*

##### Availability
* Debian unstable - Since 2024-07-02
* Debian testing - Since 2024-07-18
* Debian 12/bookworm-backports - Since 2024-08-25
* Debian 12/bookworm - Due to the mechanisms we have in place to make sure
Debian stable is in fact stable, we will never be able to ship this in the
regular repository. Users can make use of the
[backports](https://backports.debian.org/) repositories instead.
* Debian derivatives - Rolling releases will get it by the time it's on Debian
testing (e.g.: Kali Linux). Stable derivatives only in their next major release.

# The challenge
HTTP3 is fresh new, well... not really, but at least fresh enough that I'm not
aware of any other Linux distribution supporting it on curl, the reason is
likely two-fold: 

1) ### OpenSSL is not there yet  
    OpenSSL still doesn't have proper HTTP3 support, and given that OpenSSL is so
    widely used, almost every curl distributor/packager will build curl with it
    and thus changing the TLS backend to something else is risky.

    Unfortunately, proper support for the OpenSSL libcurl is unlikely to come anytime
before the end of this year, the OpenSSL performance is [not good enough
yet as of version 3.3](https://curl.se/mail/distros-2024-04/0001.html).

    Daniel Stenberg has written about the state of this multiple times, most
    recently at [HTTP/3 in curl mid
    2024](https://daniel.haxx.se/blog/2024/06/10/http-3-in-curl-mid-2024/), if
    you're interested, I suggest reading through his other posts as well.

    Some might have noticed that nginx [does support HTTP3 through OpenSSL](http://nginx.org/en/docs/quic.html),
    although when you look closely, it's not exactly perfect:  
    > An SSL library that provides QUIC support is recommended to build nginx, such as BoringSSL, LibreSSL, or QuicTLS. Otherwise, the OpenSSL compatibility layer will be used that does not support early data.

    As you can see, they don't recommend using OpenSSL, and when doing so, you don't get complete support.

2) ### HTTP3 support for GnuTLS/nghttp3/ngtcp2 is recent  
    The non-experimental support arrived [back in October
2023](https://github.com/curl/curl/commit/5f78cf503c786a1d48d13528dde038bccfa6c67c),
and so that's when I started seriously planning for this.

    curl has been working on HTTP3 support for years, and so it did support other
TLS backends before that, but out of them, the one most feasible for a
distribution to ship would be GnuTLS, which gets HTTP3 support [through ngctp2 and
nghttp3](https://daniel.haxx.se/blog/2024/06/10/http-3-in-curl-mid-2024/).

# How it was done

The Debian curl package has historically shipped at least two variants of libcurl, an
OpenSSL and a GnuTLS one.

The OpenSSL libcurl can't support HTTP3 for the reasons explained above, but
the GnuTLS libcurl can (with ngtcp2 and nghtp3).

Debian packages can choose which version of libcurl to link against (without
having to modify any upstream source code). Debian's "git" package being a famous
example of a package that links against the GnuTLS libcurl.

Enabling HTTP3 on curl was done in three steps:
1) Make sure all required dependencies fulfill the minimum requirements.
2) Enable HTTP3 for GnuTLS libcurl.
3) Change the libcurl used by the curl CLI, from OpenSSL to GnuTLS.

curl's HTTP3 support requires a somewhat recent version of nghttp3 and
updating that required a [transition](https://wiki.debian.org/Teams/ReleaseTeam/Transitions) (due to the SONAME bump), while we've also
had months of freeze for transitions due to the [time_t
transition](https://lists.debian.org/debian-devel-announce/2024/02/msg00000.html).

After the dependencies were in place, enabling HTTP3 for the GnuTLS libcurl was
[straightforward](https://salsa.debian.org/debian/curl/-/commit/51df321b0165e5164a0d898d23a64ca3bbd553c0).

Then, for the last part, we had to switch the TLS backend used by the curl CLI.
Doing the swap is also [quite
easy](https://salsa.debian.org/debian/curl/-/commit/37820dad3612d1b13a9fb9550b1726b998c80cfc)
on the packaging level, but we have to consider the chances of this change
breaking our users' environments.

# Ensuring there are no breakages

The first thing to consider regarding breakages is that this change is not
going to be pushed directly to the current Debian stable releases, it will be
present in the next stable release (13/trixie) but the current one will stick to the
version that's already shipped.

Secondly, we have to consider the risk of losing the ability to use certain
parameters from the curl CLI which could be limited to the OpenSSL backend.
During [curl-up 2024](https://daniel.haxx.se/blog/2024/05/06/i-survived-curl-up-2024/), the curl developers pointed out the existence of a page
that lists the [TLS related options and the backends they work
with](https://curl.se/libcurl/c/tls-options.html).

Analysing that page, ignoring all of the options that are suffixed with "BLOB"
(only pertinent to the library, not the CLI), the only one left which is
attention worthy is [CURLOPT_ECH](https://curl.se/libcurl/c/CURLOPT_ECH.html).

> This experimental feature requires a special build of OpenSSL, as ECH is not
> yet supported in OpenSSL releases. In contrast ECH is supported by the latest
> BoringSSL and wolfSSL releases.

As it turns out, `Encrypted Client Hello` is experimental and it's not
supported by the vanilla OpenSSL.

This was enough of an investigation for me to go ahead with the change. Noting
that even in the worst case scenario (we find a horrible regression), we can
rollback without having affected a single stable release.

Now that the package is on Debian unstable, the CI tests (autopkgtest) of every
package that depends on curl is currently running, the results are compared
against the migration-reference (in this case, the curl CLI with OpenSSL,
before the change). 

If everything goes right, curl with HTTP3 support will migrate to Debian
testing in around 5 days. If we spot any issues, we'll have to solve them
first and it's going to be hard to predict how long it takes, although it's
fair to expect less than a month.

# Feedback

Feel free to join the Matrix room for the Debian curl maintainers:  
[https://matrix.to/#/#debian-curl-maintainers:matrix.org](https://matrix.to/#/#debian-curl-maintainers:matrix.org)  

# Acknowledgements

It took us a bit longer than expected to be able to enable HTTP3, nonetheless it's
still early enough to be excited about.

A lot of people were crucial to make this happen.

I should recognize in the first place, obviously, the curl developers and the
developers of the supporting libraries: GnuTLS, nghttp3, ngtcp2. Participating
in the [curl-up
2024](https://daniel.haxx.se/blog/2024/05/06/i-survived-curl-up-2024/)
conference helped me get motivated to push this through, besides becoming aware
of the right documentation to research for impact.

On the Debian side, Sakirnth Nagarasa \<sakirnth> was responsible for updating
and taking care of the transition for nghttp3 and ngtcp2.

Also on the Debian side, I've got loads of help and support from the
co-maintainers of the curl package: Sergio Durigan Junior \<sergiodj> and Carlos
Henrique Lima Melara \<charles>.

# Changes since publication
## 2024-08-28
* Mention availability in bookworm-backports.
## 2024-07-18
* Update date of availability for Debian testing and expected date for bookworm-backports.
* Remove mention of language spoken in the Matrix room, we are using English now.
