---
layout: post
status: publish
published: true
title: 'Affirmatively Insecure: Chrome and SHA-1 Certificates'
author:
  display_name: Brian Gottreu
  login: gottreu
  email: gottreu@umn.edu
  url: ''
author_login: gottreu
author_email: gottreu@umn.edu
excerpt: SHA-1 is on the way out and being helped through the door by Google.  Brian
  will show a way to find which SSL certificates still use SHA-1 signatures and how
  to prioritize their replacement.
wordpress_id: 705
wordpress_url: http://tech.popdata.org/?p=705
date: '2015-05-31 21:46:12 -0500'
date_gmt: '2015-06-01 02:46:12 -0500'
categories:
- Dev Ops
tags:
- shaaaaaaaaaaaaa
- jq
- json
- SSL
- TLS
- CLI
comments: []
---
<p>I visited the doctor recently<sup id="fnref-705-1"><a href="#fn-705-1" rel="footnote">1<&#47;a><&#47;sup> and on the paperwork they gave me when I left was an authorization code to register with their online medical records system. I visited the page to sign up and the address bar was distressingly decorated. <img src="&#47;wp-content&#47;uploads&#47;2015&#47;05&#47;crop-insecure.png" alt="HTTPS in strikethrough with a red X over a lock" title="Affirmatively insecure" &#47;></p>
<p>If I were using an older version of Chrome, it wouldn&rsquo;t look this scary. But I&rsquo;m not, so it is. <a href="http:&#47;&#47;googleonlinesecurity.blogspot.com&#47;2014&#47;09&#47;gradually-sunsetting-sha-1.html">Starting with version 42<&#47;a>, websites using certificates that both expire on or after 2017 and use SHA-1 for signatures are considered &ldquo;affirmatively insecure&rdquo; and rendered like the preceding image. Chrome considers SHA-1 certs that expire before 2017 as &ldquo;secure, but with minor errors&rdquo;. It renders it like so. <img src="&#47;wp-content&#47;uploads&#47;2015&#47;05&#47;crop-minor-errors.png" alt="Lock with yellow triangle" title="Secure, but with minor errors" &#47;></p>
<p>If you have a SHA-1<sup id="fnref-705-2"><a href="#fn-705-2" rel="footnote">2<&#47;a><&#47;sup> cert that expires before 2016, either it will render as an ordinary secure site, or the cert is expired and the signature algorithm used is not the primary concern.</p>
<p>If you maintain websites you may wonder how you can check if your certificates use SHA-1 signatures. Halfway through writing a Perl script to do that check, I wondered that too and did a quick google search. The first result resulted in me thinking &ldquo;oh yeah&rdquo;, because it was <a href="https:&#47;&#47;shaaaaaaaaaaaaa.com&#47;">shaaaaaaaaaaaaa.com<&#47;a>, a creation of <a href="https:&#47;&#47;konklone.com&#47;">Eric Mill<&#47;a><sup id="fnref-705-3"><a href="#fn-705-3" rel="footnote">3<&#47;a><&#47;sup>, the man behind <a href="https:&#47;&#47;isitchristmas.com&#47;">isitchristmas.com<&#47;a>. I&rsquo;d visited sha{13}.com probably not that long after he <a href="https:&#47;&#47;konklone.com&#47;post&#47;why-google-is-hurrying-the-web-to-kill-sha-1">wrote about it<&#47;a>. Even though I remembered to use one of his sites to verify the Christmasness of late December days, I had forgotten about this other all-season site.</p>
<p>Not only is there the <a href="https:&#47;&#47;shaaaaaaaaaaaaa.com&#47;">shaaaaaaaaaaaaa.com<&#47;a> website where one can check if a domain uses a SHA-1 certificate, but there is also a command line <code>shaaaaaaaaaaaaa<&#47;code> tool. It&rsquo;s written in Node.js<sup id="fnref-705-4"><a href="#fn-705-4" rel="footnote">4<&#47;a><&#47;sup>, which I had never installed before. Luckily I do have homebrew installed on my Mac, so a quick <code>brew install node<&#47;code> was all that was needed for that prerequisite. Of course I already had Xcode and many other tools installed, so the process may not be as simple for you.</p>
<p>I grabbed a copy from github with<br />
<code>git clone https:&#47;&#47;github.com&#47;konklone&#47;shaaaaaaaaaaaaa.git<&#47;code>,<br />
and installed its dependencies with<br />
<code>cd shaaaaaaaaaaaaa; npm install<&#47;code>.<br />
I can check a single domain with <code>bin&#47;shaaaaaaaaaaaaa nhgis.org<&#47;code>, and get the output of</p>
<p>[code lang=javascript]<br />
{<br />
"domain": "nhgis.org",<br />
"cert": {<br />
"algorithm": "sha256",<br />
"raw": "sha256WithRSAEncryption",<br />
"good": true,<br />
"root": false,<br />
"expires": "2018-04-26T23:59:59.000Z",<br />
"name": "nhgis.org"<br />
},<br />
"intermediates": [<br />
{<br />
"algorithm": "sha384",<br />
"raw": "sha384WithRSAEncryption",<br />
"good": true,<br />
"root": false,<br />
"expires": "2024-10-05T23:59:59.000Z",<br />
"name": "InCommon RSA Server CA"<br />
},<br />
{<br />
"algorithm": "sha384",<br />
"raw": "sha384WithRSAEncryption",<br />
"good": true,<br />
"root": false,<br />
"expires": "2020-05-30T10:48:38.000Z",<br />
"name": "USERTrust RSA Certification Authority"<br />
}<br />
],<br />
"diagnosis": "good"<br />
}<br />
[&#47;code]</p>
<p>It gives the&nbsp;domain a diagnosis of "good", "bad", or "almost". A domain will get "almost" if the leaf cert uses SHA-2 but an intermediate cert still uses SHA-1.</p>
<p>I have a couple hundred domains to check, and assuming I have a file with one domain per line, then</p>
<p>[code lang=bash]<br />
cat domains.txt | xargs -P16 -L1 bin&#47;shaaaaaaaaaaaaa > output.json 2> error.json<br />
[&#47;code]</p>
<p>will do what I need. The <code>-L1<&#47;code> limits xargs to one domain per invocations of shaaaaaaaaaaaaa, and <code>-P16<&#47;code> will run up to 16 instances of shaaaaaaaaaaaaa in parallel.</p>
<p>Now I have a JSON document that says which domains use SHA-1 certs. I don&rsquo;t handle JSON all that often<sup id="fnref-705-5"><a href="#fn-705-5" rel="footnote">5<&#47;a><&#47;sup>, but I&rsquo;d heard about <a href="http:&#47;&#47;stedolan.github.io&#47;jq&#47;">jq<&#47;a> and was able to install it with homebrew. On jq's homepage it reads "jq is like sed for JSON data". Because I'm a later generation Unix nerd, I first picked up Perl for data munging and never learned the difference between awk or sed, or how to use them. But the homepage also says that jq is a command line JSON processor that lets one filter and transform JSON.<br />
To get only the domain name and diagnosis, I could do</p>
<p>[code lang=text]<br />
jq &#039;{domain: .domain, diag: .diagnosis}&#039; output.json<br />
[&#47;code]</p>
<p>To get back to the land of unstructured plain text, this would work</p>
<p>[code lang=text]<br />
jq &#039;"(.domain) (.diagnosis)"&#039; output.json|sed -e &#039;s&#47;"&#47;&#47;g&#039;<br />
[&#47;code]</p>
<p>All the SHA-1 certs should be replaced, but I want to prioritize those that might alarm users, so what I need to do is figure out which domains use SHA-1 certs that expire after Janurary 1, 2017. &nbsp;At the MPC with the certificate authority we use, when SHA-2 certs are installed to replace SHA-1 certs, new SHA-2 intermediate certs are also installed, so I won't worry about intermediate certs right now, only leaf certs. To find the "affirmatively insecure" domains, I used:<sup id="fnref-705-6"><a href="#fn-705-6" rel="footnote">6<&#47;a><&#47;sup></p>
<p>[code lang=text]<br />
cat output.json | jq &#039;select(.cert.expires >= "2017-01-01" and .cert.algorithm == "sha1") | .domain&#039;<br />
[&#47;code]</p>
<p>The output is a list of domains that recent versions of Chrome are complaining the most about and I can put those at the top of the certificate replacement list. I could do similar searches to find "secure, but with minor error" certs and the SHA-1 certs still considered secure.</p>
<p>I should point out that shaaaaaaaaaaaaa has a limited scope: determining if a domain has a certificate with SHA-1 signatures in its certificate chain. Though it does use <a href="http:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Server_Name_Indication">SNI<&#47;a>&nbsp;so you can check multiple domains hosted from a single IP, it does not check that the name on the cert actually matches the name specified. It also doesn&rsquo;t check that the cert isn&rsquo;t expired, that it&rsquo;s not self-signed, or that the server sends all the intermediate certs needed to verify that it&rsquo;s trusted by a root cert.<sup id="fnref-705-8"><a href="#fn-705-8" rel="footnote">7<&#47;a><&#47;sup></p>
<p>Perhaps if I avoid googling "tools to check certificate validity", I&rsquo;ll extend my Perl script to check for all those problems and write about it too.<sup id="fnref-705-9"><a href="#fn-705-9" rel="footnote">8<&#47;a><&#47;sup></p>
<div class="footnotes">
<hr &#47;>
<ol>
<li id="fn-705-1">
&nbsp;Do not worry, neither my lungs nor my posterior had been poisoned with poisonous gases. &nbsp;Enjoy <a href="https:&#47;&#47;www.youtube.com&#47;watch?v=B1BdQcJ2ZYY">some clips from a Flight of the Conchords episode<&#47;a>.&#160;<a href="#fnref-705-1" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-2">
While typing "a SHA-1", I realized that Google pronounces it as a single syllable followed by a number and not as three letters and a number.&#160;<a href="#fnref-705-2" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-3">
Eric Mill is also the only person that Snapchats with me. The stream is 90%+ cats and 100% delightful.&#160;<a href="#fnref-705-3" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-4">
I once had a discussion if it&rsquo;s &ldquo;in Node&rdquo; or &ldquo;on Node&rdquo;. I don&rsquo;t recall what the consensus was.&#160;<a href="#fnref-705-4" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-5">
Most of what I deal with is still plain text with whitespace delimiters. Also, get off my lawn.&#160;<a href="#fnref-705-5" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-6">
We&rsquo;re comparing the dates here as strings, but the dates are in ISO 8601 format, so it&rsquo;ll be fine as long as none of the expiration times use a non-UTC timezone. Pertinent: <a href="https:&#47;&#47;xkcd.com&#47;1179&#47;">xkcd.com&#47;1179<&#47;a> Also, complain about the <a href="http:&#47;&#47;www.smallo.ruhr.de&#47;award.html">useless use of cat<&#47;a> someplace else.&#160;<a href="#fnref-705-6" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-8">
Not to mention whether the server allows weak protocols or ciphers, or if it&rsquo;s vulnerable to Heartbleed, POODLE, FREAK, or Narnia.&#160;<a href="#fnref-705-8" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<li id="fn-705-9">
I could improve shaaaa to do those things, but 1) Mill doesn&rsquo;t wish to expand the scope of the tool, 2) all the Node SSL libraries I&rsquo;ve checked seem to use the openssl command line tool and mangle its output rather than interface with the C library, 3) I&rsquo;m more fluent in Perl than Javascript.&#160;<a href="#fnref-705-9" rev="footnote">&#8617;<&#47;a><br />
<&#47;li></p>
<p><&#47;ol><br />
<&#47;div></p>
