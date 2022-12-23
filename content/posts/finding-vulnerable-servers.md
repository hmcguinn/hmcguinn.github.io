---
title: "Finding Vulnerable DoD Apache Tomcat Servers"
date: 2024-08-09T20:00:00-05:00
draft: true
---

# Introduction

The Department of Defense launched their [vulnerablity disclosure program](https://hackerone.com/deptofdefense) (VDP) in 2016. I've been interested in bug bounty/VDPs for awhile but never actually gave them a try. I chose to investigate this particular program because I thought it would be relatively easier to find a vulnerability in a huge legacy organization like the DoD. And it worked! In a couple of hours I was able to find, validate, and submit my first vulnerability.

# Reconnaissance

## IP Blocks

The DoD lists the scope of their VDP as ``"Publicly accessible information systems, web property, or data owned, operated, or controlled by DoD."`` This is a pretty big scope! Where do I even start? Attack surface management tools like [Shodan](https://www.shodan.io/) and [Censys](https://search.censys.io/) let us quickly get an overview of publically facing servers and what they are running. Personally, I prefer Censys because it is easier to filter fields and has higher quality data (full disclosure: I interned there in 2021). I knew that the DoD has huge CIDR blocks, so I looked up their [/8s on Wikipedia](https://en.wikipedia.org/wiki/List_of_assigned_/8_IPv4_address_blocks#List_of_assigned_/8_blocks_to_the_United_States_Department_of_Defense).

I started by searching for all active servers in their IP blocks. For example, [55.0.0.0/8](https://search.censys.io/search?resource=hosts&sort=RELEVANCE&per_page=25&virtual_hosts=EXCLUDE&q=ip%3A+55.0.0.0%2F8). Unfortunately, I didn't get any results with this method. I know that a lot of the IP blocks were unused, so it could be that I was unlucky or just that Censys doesn't support searching on an entire /8. I'd have to figure out another way to find some targets.

## Certificate Names

Luckily, Censys also supports searching on certificates and HTTP headers. I can search for all servers presenting a ``*.mil`` (top-level domain for the DoD) certificate using [this](https://search.censys.io/search?resource=hosts&sort=RELEVANCE&per_page=25&virtual_hosts=EXCLUDE&q=services.tls.certificates.leaf_data.subject.common_name%3A+*.mil) query. I got back around 30,000 results, which is pretty hard to exhaustively comb through.

I started by filtering for exposed database protocols: ``MYSQL``, ``MongoDB``, etc. Exposing these without proper authentication is no-bueno. My quick scan didn't reveal anything that I thought was worth investigating further so I continued on. Unfortunately, I didn't keep the best records of the exact queries I used at this point. The basic process was to search for a specific service, investigate, pivot to a new common name or organization. Eventually I stumbled onto an [Apache Tomcat](https://tomcat.apache.org/) server running in AWS GovCloud that looked promising.


# Scanning

Coincidentally, I had worked on a [HackTheBox](https://arkanoidctf.medium.com/hackthebox-writeup-jerry-aa2b992917a7) that involved an Apache Tomcat server so I have a little familiarity in this area. Visiting the server's page, I was greeted with this:

![homepage](/img/website.png)

Pretty standard for an unconfigured server, but the version ``8.0.21`` is promising. A quick google reveals that Tomcat 8.0.x reached [end-of-life](https://tomcat.apache.org/tomcat-80-eol.html) in **June 2018!** 

The next step was to check for CVEs against this version. Turns out there are more than a few! Tomcat 8.0.21 has [16 published CVEs](https://www.cvedetails.com/vulnerability-list.php?vendor_id=45&product_id=887&version_id=540778&page=1&hasexp=0&opdos=0&opec=0&opov=0&opcsrf=0&opgpriv=0&opsqli=0&opxss=0&opdirt=0&opmemc=0&ophttprs=0&opbyp=0&opfileinc=0&opginf=0&cvssscoremin=0&cvssscoremax=0&year=0&month=0&cweid=0&order=3&trc=16&sha=25e452bacc35f15fb25b5d9df0121acfe6a4f72b). This server definitely should be updated. But I needed to confirm it's actually vulnerable to these before I made a report.

# Confirming Vulnerability

Tomcat 8.0.21 is vulnerable to [CVE-2015-5345](https://nvd.nist.gov/vuln/detail/CVE-2015-5345), a path traversal vulnerability. Luckily, it's pretty easy to test as well. The full details of the vulnerability are [here](https://hackdefense.com/publications/cve-2015-5345-apache-tomcat-vulnerability/).

Confirming the server's vulnerability is as simple as running: ``curl -I -k <IP>/WEB-INF``

```
➜  curl -I -k https://<IP>:7443/WEB-INF

HTTP/1.1 302 Found
Date: Sat, 11 Jun 2022 19:39:17 GMT
Server: Apache
X-Distributed-by: The Apache Haus (http://www.apachehaus.com)
Location: https://<IP>:7443/WEB-INF/
Content-Type: text/plain
```

Due to the bug, the server improperly responds with a ``302 Found`` instead of a ``404 Not Found``. The proper behaviour can be seen by running: ``curl -I -k <IP>/WEB-INF/``

```
➜  curl -I -k https://<IP>:7443/WEB-INF/

HTTP/1.1 404 Not Found
Date: Sat, 11 Jun 2022 19:39:55 GMT
Server: Apache
X-Distributed-by: The Apache Haus (http://www.apachehaus.com)
Content-Language: en
Content-Length: 992
Content-Type: text/html;charset=utf-8

```

We've now confirmed that the server is actually vulnerable to CVE-2015-5345. At this point, I was preparing to write up my findings and submit them to the VDP. 


# Pivoting

Before I submitted, I did one last search in Censys for the certificate the server was presenting, with common name: ``<REDACTED>.us.army.mil``. I got two results, the server I had been investigating and a seemingly identical server with a different IP. 

This other server was also running Tomcat 8.0.21 and I quickly confirmed it was also vulnerable.

# Reporting

I wrote up my findings and submitted them to the VDP Saturday afternoon. I heard back early on Monday morning and was asked to create two separate reports for each server. After updating and submititting new reports, they were both triaged and validated within about 30 minutes. Huge props to the DoD for being so quick!


# Lessons

Overall, this was a low severity vulnerability. The specific CVE I confirmed is relatively low/medium impact and the two servers didn't appear to be in active use. Still, it's not good to have unpatched, end-of-life servers publically exposed from your networks.

Working on this was a ton of fun! Hopefully I'll have more findings to report in the future!
