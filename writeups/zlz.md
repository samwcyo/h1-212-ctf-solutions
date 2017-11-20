Process
The first thing I did when attempting this CTF was gather information about the host. After running an nmap scan against the network I was able to determine that both 80 and 22 were open and accessible.

Since port 22 was open (and it was the only other port open besides 80) I decided to attempt to connect to it in order to check and see whether or not I could login with a username and password. After it prompted me to enter a username and password I was able to determine that later in the challenge I would potentially have to login with stumbled across credentials, etc. After completing the CTF I realized this was not the case, but it was still a step I took in fingerprinting the system.

Since the SSH information returned from nmap did not declare it directly vulnerable to any sort of attack that could be directly carried out over port 22 I simply ignored that it was open as I switched my focus to port 80.

Normally when attempting a web application CTF the first thing I look for is any sort of hidden information in the source code or point of interaction. With this challenge, however, there was none. Using a Google dork I was able to pull the same Apache default page and compare it line-by-line to the file hosted on the server. After seeing that they were in fact the exact same I decided that I would either have to (1) guess the directory or (2) modify something else in the request to allow me to interact with another portion of the site (accept, host, content-type, request method, etc).

At this point Jobert had already tweeted hinting that directory brute forcing was basically pointless, so I went ahead and started modifying other stuff in the request.

One of the things that caught my attention about the initial tweet was the fact that a domain was provided even thought we were talking to an IP address.

Whenever I modified the host header (e.g. acme.org, google.com, etc) the only real result was it being reflected in error messages. For instance if we visit http://104.236.20.43/abc with a modified host header to "google.com" it will say "Apache/2.4.18 (Ubuntu) Server at google.com Port 80". This was somewhat insightful because it meant that the server was processing the host header (there are some cases where cloudflare is in use and will prevent any modification of this unless the domain is registered under it).

There was a little bit of enumeration but eventually I eventually realized that a different page was presented when I sent "admin.acme.org" as the host header. The reason I tried this was because the prompt that (1) it was a new admin panel, (2) developed for acme.org.

The response for "/" at host header "admin.acme.org" returned "Set-cookie: admin=no" so I immediately changed my request to include "Cookie: admin=yes" and the behavior it displayed was different than any behavior present with "admin=no" or no cookie whatsoever. The initial response with "admin=yes" was "405 method not allowed".

Initially I thought the 405 method was some sort of troll by the CTF developers saying that I needed to find another way of becoming admin (e.g. "user=no" v.s. "admin=yes") but it turns out it was pretty much the literal 405 error definition of "GET" not being allowed.

After this occurred I switched my methods around. I tried POST and received "406", OPTIONS and received "405". It was strange that POST gave "406" so I went back to investigate what it was actually doing. After speaking with someone on Slack and reading the #CTF conversation I came to realize that the "406" was actually just a message by the CTF developers saying "you're on the right track - dig deeper" so I continued to modify my request with the POST method.

One of the things I immediatly started modifying was the "content type" header. After setting it to application/xml - no luck. After setting it to application/json - hey! Some weird response! "418 I'm a teapot" and "unable to decode" in the response body.

The most immediate thing I did after seeing "cannot decode" was send the following JSON formatted POST body and added 'content-length' header:
{} (very simple! yes!)

After sending the request it responded with "required domain value", so I sent the following request:
{"domain":"google.com"}

After sending the request it told me that it required a domain with ".com". What? My domain had .com!

This part honestly got me for a long time. Eventually I sent "212.acme.com" since the challenge was (1) relative to 212 and (2) kept telling me that it was an 'incorrect value' (as if I had to supply some sort of password type thing) and it responded with "{"next":"\/read.php?id=0"}".

After visiting "/read.php?id=0" and seeing the response "data" (blank) I thought I knew what was going on -- this was some sort of SSRF with responses or potentially a local file inclusion.

I revisited a few articles about SSRF (thank you Orange Tsai!) and found out that you were able to send "\" but pretty much no other control character besides slashes.

One of the payloads from a presentation seemed to work for this request as "localhost\n\r212.google.com" returned the base64 encoded content for the local default apache page. This meant that I could send requests through the network to the local network.

The reason I think it would send requests to localhost is because it handled the request as a "for each entry in the array of items separated by a new line do this" because it would increment the "read.php?id=x" by two each time: one to the dummy website we specified to bypass the filter and one to the localhost endpoint we specified.

After a little while playing with it I realized that I could fetch the response from ports (the SSH banner being the  prime example) and additionally view local content that was 403 for anyone else (the server-status page).

After viewing the server status page I thought to myself: the answer MUST be hidden in the "tripwire" function the server has! I spent a ton of time googling for the "tripwire" module that apache has only to find that there was no web application/panel for it.

I had installed a local version of tripwire using "apt-get install tripwire" but really couldn't find anything. Back to the other issue... I could read from ports.

Since my scan only returned 80 and 22 being open (and nothing being filtered) I was a little bummed. Maybe there were open ports that I couldn't see on localhost?

After a bit of scanning and messing around (I was super close to just writing an extension or some sort of copy->paste website that would format the responses automatically) I saw that port 1337 returned content that was different than anything I'd seen!

Hmm, where would it be?

Probably in... /flag?

FLAG: CF,2dsV\/]fRAYQ.TDEp`w"M(%mU;p9+9FD{Z48X*Jtt{%vS($g7\S):f%=P[Y@nka=<tqhnF<aq=K5:BC@Sb*{[%z"+@yPb/nfFna<e$hv{p8r2[vMMF52y:z/Dh;{6

Hey! The flag! WOOT! What a headache... this was the second punch in the face of obscurity after trying to solve Jobert's "anti-ctf" ctf's.

SUPER COOL CHALLENGE: the only part that bothered me was the fact the error was so strange for the domain. I guess that sort of thing happens in the wild, though!
