+++
title = "Announcing wcurl: a curl wrapper to download files"
date = 2024-07-03
draft = false

[taxonomies]
categories = ["debian", "curl"]

[extra]
toc = true
thumbnail = "announcing_wcurl_image.png"
+++

{{ youtube(id="eM8M5qa4pPM") }}

#### tl;dr

Whenever you need to download files through the terminal and don't feel like using wget:  
```console
wcurl example.com/filename.txt
```

Manpage:  
[https://manpages.debian.org/unstable/curl/wcurl.1.en.html](https://manpages.debian.org/unstable/curl/wcurl.1.en.html)

##### Availability (comes installed with the curl package):
* Debian unstable - Since 2024-07-02
* Debian testing - Since 2024-07-18
* Debian 12/bookworm-backports - Since 2024-08-25
* Debian 12/bookworm - Depends on whether Debian's release team will approve
it, it could be available in the next point release.
* Debian derivatives - Rolling releases will get it by the time it's on Debian
testing (e.g.: Kali Linux). Stable derivatives only in their next major release.

If you don't want to wait for the package update to arrive, you can always copy
the script and place it in your `/usr/bin`, the code is here:  
[https://github.com/Debian/wcurl/blob/main/wcurl](https://github.com/Debian/wcurl/blob/main/wcurl)  
[https://salsa.debian.org/debian/wcurl/-/blob/main/wcurl?ref_type=heads](https://salsa.debian.org/debian/wcurl/-/blob/main/wcurl?ref_type=heads)

# Smoother CLI experience

Starting with **curl version 8.8.0-2**, the Debian's curl package now ships a wcurl
executable.

wcurl is the solution for those who just need to download files without having
to remember curl's parameters for things like automatically naming the files.

Some people, myself included, would fall back to using wget whenever there was a
need to download a file. Sometimes even installing wget just for that usecase.
After all, it's easier to remember "apt install wget" rather than "curl -L -O -C - ...".

wcurl consists of a simple shell script that provides sane defaults for the curl
invocation, for when the use case is to just download files.

By default, wcurl will:  
> * Encode whitespaces in URLs;
> * Download multiple URLs in parallel if the installed curl's version is >= 7.66.0;
> * Follow redirects;
> * Automatically choose a filename as output;
> * Avoid overwriting files if the installed curl's version is >= 7.83.0 (--no-clobber);
> * Perform retries;
> * Set the downloaded file timestamp to the value provided by the server, if available;
> * Default to the protocol used as https if the URL doesn't contain any;
> * Disable curl's URL globbing parser so {} and [] characters in URLs are not treated specially.

Example to download a single file:
```console
wcurl example.com/filename.txt
```

If you ever need to set a custom flag, you can make use of the `--curl-options`
wcurl option, anything set there will be passed to the curl invocation.
Just beware that if you need to set any custom flags, it's likely you will be
better served by calling curl directly. The `--curl-options` option is there to
allow for some flexibility in unforeseen circumstances.

# The need for wcurl

I've always felt a bit ashamed of not remembering curl's parameters for
downloading a file and automatically naming it, having resorted to wget most of
the times this was needed (even installing wget when it wasn't there, just for
this). I've spoken to a few other experienced people I know and confirmed what
could be obvious to others: a lot of people struggle with this.

Recently, the curl project released the results of [2024's curl
survey](https://daniel.haxx.se/media/curl-user-survey-2024-analysis.pdf), which
also showed this is as a much needed feature, just look at some of the answers:

#### Q: Which curl command line option do you think needs improvement and how?

> -O, I really want wget like functionality where I don't have to specify the name

> Downloading a file (like wget) could be improved - with automatic naming of the file

> downloading files - wget is much cleaner

> I wish the default behaviour when GETting a binary was to drop it on disk. That's the only
reason 'wget foo.tgz" is still ingrained in my muscle memory .

> Maybe have a way to download without specifying something in -o (the only reason i used wget
still)

> --remote-time should be default

> --remote-name-all could really use a short flag

#### Q: If you miss support for something, tell us what!

> "Write the data to the file named in the URL (or in redirects if I'm feeling daring), and
timestamp the file to the last-modified-date". This is the main reason I'm still using wget.

I can finally feel less bad about falling back to wget due to not remembering the
parameters I want.

# Idealization vs. reality

I don't believe curl will ever change its default behavior in such a way that
would accommodate this need, as that would have a side-effect of breaking things
which expect the current behavior (the blast radius is literally the
[solar system](https://daniel.haxx.se/blog/2021/12/03/why-curl-is-used-everywhere-even-on-mars/)).

This means a new executable needs to be shipped side-by-side with curl, an
opportunity to start fresh and work with a more focused use case (to download
files).

Ideally, this new executable would be maintained by the curl project, make use
of libcurl under-the-hood, and be available everywhere. Nobody wants to worry
if their systems have the tool or not, it should always be there.

Given I'm just a Debian Developer, with not as much free time as I wish, I've
decided to write a simple shell script wrapper calling the curl CLI
under-the-hood. 

wcurl will come installed with the curl package from now on, and I will check
with the release team about shipping it on the current Debian stable as well.
Shipping wcurl in other distros will be up to them (Debian-derivatives should
pick it up automatically, though).

We've tried to make it easy for anyone to ship this by using the curl license,
keeping the script POSIX-compliant, and shipping a [manpage](https://manpages.debian.org/unstable/curl/wcurl.1.en.html).

Maybe if there's enough interest across distributions, someone might sign up
for implementing this in upstream curl and increase its reach. I would be happy
with the curl project reusing the wcurl name when that happens. It's unlikely
that wcurl would be shipped by curl upstream as it is, assuming they would
prefer a solution that uses libcurl direclty (more similar to curl the CLI, to
maintain).

In the worst case, wcurl becomes a Debian-specific tool that only a few people
are aware of, in the best case, it becomes the new go-to CLI tool for simply
downloading files. I would be happy if at least someone other than me finds
it useful.

# Naming is hard

When I started working on it, I was calling the new executable "curld"
(stands for "curl download"), but then when discussing this in one of our
weekly calls in the Debian Brasília community, it was mentioned that this could
be confused for a daemon.

We then settled for the name "wcurl", suggested by Carlos Henrique Lima
Melara \<charles>. It doesn't really stand for anything,
but it's very easy to remember.  

You know... "it's that wget alternative for when you want to use curl instead"
:)

# Feedback

I'm hosting the code on Github and Debian's GitLab instance, feel free to open an issue to provide feedback.  
[https://salsa.debian.org/debian/wcurl](https://salsa.debian.org/debian/wcurl)  
[https://github.com/Debian/wcurl](https://github.com/Debian/wcurl)

We also have a Matrix room for the Debian curl maintainers:  
[https://matrix.to/#/#debian-curl-maintainers:matrix.org](https://matrix.to/#/#debian-curl-maintainers:matrix.org)  

# Acknowledgments

The idea for wcurl came a few days before the [curl-up conference
2024](https://daniel.haxx.se/blog/2024/05/06/i-survived-curl-up-2024/).
I've been thinking a lot about developer productivity in the terminal lately,
different tools and better defaults. Before curl-up, I was also thinking about
packaging improvements for the curl package. I don't remember what exactly
happened, but I likely had to download something and felt a bit ashamed of
maintaining curl and not remembering the parameters to download files the way I
wanted.

I first discussed this idea in the conference, where I asked the
participants about it and there were no concerns raised, and some people said I should give it a go.
Participating in curl-up was a really great experience and I'm thankful for the
interactions I've had there.

On the Debian side, I've got reviews of the code and manpage by Sergio Durigan
Junior \<sergiodj>, Guilherme Puida Moreira \<puida> and Carlos Henrique Lima
Melara \<charles>. Sergio ended up rewriting the tool to be POSIX-compliant (my
version was written in bash), so he takes all the credit for the portability.

# Changes since publication
## 2024-07-18
* Update date of availability for Debian testing and expected date for bookworm backports.
* Mention charles as the person who suggested "wcurl" as a name.
* Update wcurl's -o/--opts options, it's now just --curl-options.
* Remove mention of language spoken in the Matrix room, we are using English now.
* Update list of features of wcurl.
## 2024-08-28
* Mention availability in bookworm-backports.
* Link to wcurl lightning talk from DebConf24.
