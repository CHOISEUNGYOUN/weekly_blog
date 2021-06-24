## Understanding "same-site" and "same-origin"

"same-site" and "same-origin" are frequently cited but often misunderstood terms. For example, they are mentioned in the context of page transitions, fetch() requests, cookies, opening popups, embedded resources, and iframes.

### Origin
![alt origin](imgs/Understanding-same-site-and-same-origin/1.png)

"Origin" is a combination of a [scheme](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Identifying_resources_on_the_Web#scheme_or_protocol) (also known as the [protocol](https://developer.mozilla.org/en-US/docs/Glossary/Protocol), for example [HTTP](https://developer.mozilla.org/en-US/docs/Glossary/HTTP) or [HTTPS](https://developer.mozilla.org/en-US/docs/Glossary/HTTPS)), [hostname](https://en.wikipedia.org/wiki/Hostname), and [port](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Identifying_resources_on_the_Web#port) (if specified). For example, given a URL of `https://www.example.com:443/foo` , the "origin" is `https://www.example.com:443`.

### "same-origin" and "cross-origin"
Websites that have the combination of the same scheme, hostname, and port are considered "same-origin". Everything else is considered "cross-origin".


|Origin A|Origin B|Explanation of whether Origin A and B are "same-origin" or "cross-origin"|
|--------|--------|-------------------------------------------------------------------------|
|https://www.example.com:443|https://www.evil.com:443 <br> https://example.com:443 <br>	https://login.example.com:443 <br> http://www.example.com:443 <br> https://www.example.com:80	<br> https://www.example.com:443 <br> https://www.example.com| cross-origin: different domains <br> cross-origin: different subdomains <br>	cross-origin: different subdomains <br> cross-origin: different schemes <br> cross-origin: different ports <br> same-origin: exact match <br> same-origin: implicit port number (443) matches|

### Site
![alt origin](imgs/Understanding-same-site-and-same-origin/2.png)

Top-level domains (TLDs) such as `.com` and `.org` are listed in the [Root Zone Database](https://www.iana.org/domains/root/db). In the example above, "site" is the combination of the TLD and the part of the domain just before it. For example, given a URL of `https://www.example.com:443/foo` , the "site" is `example.com`.

However, for domains such as `.co.jp` or `.github.io`, just using the TLD of `.jp` or `.io` is not granular enough to identify the "site". And there is no way to algorithmically determine the level of registrable domains for a particular TLD. That's why a list of "effective TLDs"(eTLDs) was created. These are defined in the [Public Suffix List](https://wiki.mozilla.org/Public_Suffix_List). The list of eTLDs is maintained at [publicsuffix.org/list](https://publicsuffix.org/list/).

The whole site name is known as the eTLD+1. For example, given a URL of `https://my-project.github.io` , the eTLD is `.github.io` and the eTLD+1 is `my-project.github.io`, which is considered a "site". In other words, the eTLD+1 is the effective TLD and the part of the domain just before it.

![alt origin](imgs/Understanding-same-site-and-same-origin/3.png)

### "same-site" and "cross-site"
Websites that have the same eTLD+1 are considered "same-site". Websites that have a different eTLD+1 are "cross-site".

|Origin A|Origin B|Explanation of whether Origin A and B are "same-site" or "cross-site"|
|--------|--------|-------------------------------------------------------------------------|
|https://www.example.com:443|https://www.evil.com:443 <br>	https://login.example.com:443 <br> http://www.example.com:443 <br> https://www.example.com:80	<br> https://www.example.com:443 <br> https://www.example.com| cross-site: different domains <br> same-site: different subdomains don't matter <br> same-site: different schemes don't matter <br> same-site: different ports don't matter <br> 	same-site: exact match <br> same-site: ports don't matter|

![alt origin](imgs/Understanding-same-site-and-same-origin/4.png)

### "schemeful same-site"

The definition of "same-site" is evolving to consider the URL scheme as part of the site in order to prevent HTTP being used as [a weak channel](https://datatracker.ietf.org/doc/html/draft-west-cookie-incrementalism-01#page-8). As browsers move to this interpretation you may see references to "scheme-less same-site" when referring to the older definition and ["schemeful same-site"](https://web.dev/schemeful-samesite/) referring to the stricter definition. In that case, `http://www.example.com` and `https://www.example.com` are considered cross-site because the schemes don't match.

|Origin A|Origin B|Explanation of whether Origin A and B are "schemeful same-site"|
|--------|--------|-------------------------------------------------------------------------|
|https://www.example.com:443|https://www.evil.com:443 <br>	https://login.example.com:443 <br> http://www.example.com:443 <br> https://www.example.com:80	<br> https://www.example.com:443 <br> https://www.example.com| cross-site: different domains <br> schemeful same-site: different subdomains don't matter <br> cross-site: different schemes <br> schemeful same-site: different ports don't matter <br> schemeful same-site: exact match <br> chemeful same-site: ports don't matter|

### How to check if a request is "same-site", "same-origin", or "cross-site"

Chrome sends requests along with a `Sec-Fetch-Site` HTTP header. No other browsers support `Sec-Fetch-Site` as of April 2020. This is part of a larger [Fetch Metadata Request Headers](https://www.w3.org/TR/fetch-metadata/) proposal. The header will have one of the following values:

* `cross-site`
* `same-site`
* `same-origin`
* `none`

By examining the value of `Sec-Fetch-Site`, you can determine if the request is "same-site", "same-origin", or "cross-site" ("schemeful-same-site" is not captured in `Sec-Fetch-Site`).

추가 참고 사항:

Original Source:
[Understanding "same-site" and "same-origin"](https://web.dev/same-site-same-origin/)
