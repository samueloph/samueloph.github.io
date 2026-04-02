+++
title = "Bringing HTTP/3 to curl on Amazon Linux"
date = 2026-04-02
draft = false

[taxonomies]
categories = ["amazonlinux", "curl"]

[extra]
toc = true
thumbnail = "bringing-http3-to-curl-on-amazon-linux.webp"
+++

{{ img(src="./bringing-http3-to-curl-on-amazon-linux.webp" alt="Screenshot of
the top entry of the curl package's changelog, showing the following:
Changelogs for curl-8.17.0-1.amzn2023.0.2.x86_64
* Mon Mar 16 00:00:00 2026 Samuel Henrique (samueloph) <samhn@amazon.com> - 8.17.0-1.amzn2023.0.2
- Enable HTTP/3 support in the full build using ngtcp2 and nghttp3
- HTTP/3 is explicitly disabled in the minimal build
- Add runtime dependencies on libnghttp3 and libngtcp2 with minimum version pinning
- Run tests in parallel via upstream make test-nonflaky, with serial fallback for race-prone tests") }}

#### tl;dr

Starting with **curl 8.17.0-1.amzn2023.0.2** in Amazon Linux 2023, you can now use HTTP/3.
```bash
dnf swap -y libcurl-minimal libcurl-full
dnf swap -y curl-minimal curl-full
curl --http3-only https://example.com
```
*(HTTP/3 is only enabled in the curl -full builds)*

Or, if you would like to try it out in a container:
```bash
podman run amazonlinux:2023 /bin/sh -c 'dnf upgrade -y --releasever=latest && dnf swap -y libcurl-minimal libcurl-full && dnf swap -y curl-minimal curl-full && curl --http3-only https://example.com'
```

For a list of test endpoints, you can refer to
[https://bagder.github.io/HTTP3-test/](https://bagder.github.io/HTTP3-test/)

# The Upgrade I Didn't Have to Make

My teammate Steve Zarkos, who previously worked on upgrading OpenSSL in Amazon
Linux from 3.0 to 3.2, spent the last few months on the complex task of bumping
OpenSSL again, this time to 3.5. A bump like this only happens after extensive
code analysis and testing, something that I didn't foresee happening when
AL2023 was released but that was a notable request from users.

Having [enabled HTTP/3 on
Debian](https://samueloph.dev/blog/debian-curl-now-supports-http3/), I was
always keeping an eye on when I would get to do the same for Amazon Linux (mind
you, I work at AWS, in the Amazon Linux org). The bump to OpenSSL 3.5 was the
perfect opportunity to do that, for the first time Amazon Linux is shipping an
OpenSSL version that is supported by ngtcp2 for HTTP/3 support.

# Non-Intrusive Change

In order to avoid any intrusive changes to existing users of AL2023, I've only
enabled HTTP/3 in the full build of curl, not in the minimal one, this means
there is no change for the minimal images.

The way curl handles HTTP/3 today also does not lead to any behavior changes
for those who have the full variants of curl installed, this is due to the fact
that HTTP/3 is only used if the user explicitly asks for it with the flags
`--http3` or `--http3-only`.

# Side Quests

Supporting HTTP/3 on curl also requires building it with ngtcp2 and nghttp3,
two packages which were not shipped in Amazon Linux, besides, my team doesn't
even own the curl package, we are a security team so our packages are the
security related stuff such as OpenSSL and GnuTLS. Our main focus is the
services behind Amazon Linux's vulnerability handling, not package maintenance.

I worked with the owners of the curl package and got approvals on a plan to
introduce the two new dependencies under their ownership and to enable the
feature on curl, I appreciate their responsiveness.

Amazon Linux 2023 is forked from Fedora, so while introducing ngtcp2, I also
sent a couple of Pull Requests upstream to keep things in sync:

[[ngtcp2] package latest release 1.21.0](https://src.fedoraproject.org/rpms/ngtcp2/pull-request/9)

[[ngtcp2] do not skip tests](https://src.fedoraproject.org/rpms/ngtcp2/pull-request/8)

While building the curl package in Amazon Linux, I've noticed the build was
taking 1 hour from start to end, and the culprit was something well known to
me; tests.

The curl test suite is quite extensive, with more than 1600 tests, all of that
running without parallelization, running two times for each build of the
package; once for the minimal build and again for the full build.

I had previously enabled parallel tests in Debian back in 2024 but never got
around to submit the same improvements to Amazon Linux or Fedora, this is now
fixed. The build times for Amazon Linux came down to 10 minutes under the same
host (previously 1 hour), and Fedora promptly merged my PR to do the same
there:

[[curl] run tests in parallel](https://src.fedoraproject.org/rpms/curl/pull-request/80)

All of this uncovered a test which is timing-dependent, meaning it's not
supposed to be run with high levels of parallelism, so there goes another PR,
this time to curl:

[Flag test 766 as timing-dependent#21155](https://github.com/curl/curl/pull/21155)

What started as enabling a single feature turned into improvements that landed
in curl, Fedora, and Amazon Linux alike. I did this in a mix of work and
volunteer time, mostly during work hours (work email address used when this was
the case), but I'm glad I put in the extra time for the sake of improving curl
for everyone.

# Release Notes
[Amazon Linux 2023 release notes for 2023.10.20260330](https://docs.aws.amazon.com/linux/al2023/release-notes/relnotes-2023.10.20260330.html)
