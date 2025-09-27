
<!-- vim: set foldmethod=marker fmr=###,--- :-->

*Throughout this page, replace ALL CAPS with correct values*

#### General Notes

For sending mail, the DOMAIN is the main domain — we don't use subdomains.

This can be confusing, because for DKIM we use **selectors** — a selector is just any word that shows which DKIM record is being referenced.

#### ★ start FTP-ing SYNC folders now
---
</details><details><summary>Postfix Configuration</summary>

### Postfix Configuration

Edit the Postfix configuration file:
```
vi /etc/postfix/main.cf
```
Update these lines (replace DOMAIN with domain.tld):
```
mydestination =                   # line 35
myhostname = DOMAIN               # line 87
inet_interfaces = loopback-only   # line 88
```
Paste at the end:
```
milter_protocol = 6
milter_default_action = accept
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```
----------

</details><details><summary>OpenDKIM Configuration</summary>

### OpenDKIM Configuration

The `SELECTOR` can be any string; it is used to create a corresponding `DKIM` DNS record.

We use the name of the server.
```
vi /etc/opendkim.conf
```
```
14 Mode             sv                                 # uncomment
15 SubDomains       no                                 # uncomment

22 Domain           DOMAIN                             # uncomment & update
23 Selector         SELECTOR                           # uncomment & update
23 KeyFile          /etc/dkimkeys/SELECTOR.private     # uncomment & update

31 UMask            002                                # update

37 #Socket          local:/run/dkimkeys/dkimkeys.sock  # comment
38 Socket           inet:8891@localhost                # uncomment

46 InternalHosts    127.0.0.1, ::1, localhost          # uncomment & update
```
Add the following lines at the end:
```
AutoRestart      yes
AutoRestartRate  10/1h
Background       yes
RequireSafeKeys  False
```
Copy the following into a text editor and update ALLCAPS text before running:
```
opendkim-genkey -b 1024 -D /etc/dkimkeys -s SELECTOR -d DOMAIN -v
chown opendkim:opendkim /etc/dkimkeys -R
chmod 700 /etc/dkimkeys
chmod 600 /etc/dkimkeys/*.private
```
----

</details><details><summary>DNS Records</summary>

### DNS Records

**THREE DNS Records**

```
cat /etc/dkimkeys/*.txt
```
1. **DKIM**: with the the output, create a `TXT` record with hostname `SELECTOR._domainkey`
```
v=DKIM1; h=sha256; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQD05TN1MPYCoMecpK...
```
2. **SPF**: create a `TXT` record with an empty hostname  
   — to include many IP addresses, just separate them with returns  
   — do not use quotes (at Linode — other DNS services may vary)
```
v=spf1
mx include:_spf.google.com                            # if necessary
ip4:000.000.000.000 ip6:0000:0000:0000:0000:0000:0000
ip4:000.000.000.000 ip6:0000:0000:0000:0000:0000:0000 # if more than one
-all
```
[Google MX Toolbox](https://toolbox.googleapps.com/apps/checkmx/)

3. **DMARC**: create a `TXT` record with hostname `_dmarc`:
```
v=DMARC1; p=quarantine; rua=mailto:EMAIL; ruf=mailto:EMAIL; sp=none; aspf=r; adkim=r
```
Notes:
- there can only be one `v=spf1` record, so merge all domains/ip addresses
- `p=quarantine` → suspicious mail goes to spam   
  switch later to `p=reject` after confirmation that everything works
- `rua/ruf` = addresses where you’ll get DMARC reports  
  name+domain+dmarc@example.com works well
```
systemctl restart opendkim
systemctl restart postfix
```
A new site can now be created: [github.com/svijasvg/site-mgmt](https://github.com/svijasvg/site-mgmt)

---

</details><details><summary>Testing</summary>

### Testing

Send test messages:
```
echo "Test mail from $(hostname)" | mail -s "Test" EMAIL # or
swaks --to EMAIL --from noreply@DOMAIN --server localhost
```
Check if the mail queue has been emptied:
```
mailq
```
To see if the email has been correctly configured:
1. open the message in Gmail
2. click on the ⋮ next to the reply arrow and select "< > Show original"
3. look at the section called `ARC-Authentication-Results:`  
   you should see:
```
       dkim=pass ...
       spf=pass ...
```
In general, DKIM signing will work immediately but that the SPF records need several hours to take effect.

---

</details><details><summary>Debugging</summary>

### Debugging

To reconfigure postfix if it was wrongly configured:
```
# must use sudo
sudo dpkg-reconfigure postfix
```
- port `25` needs to be open on the server (it may be blocked by default on linodes)

To check:
```
telnet smtp.gmail.com 25
```
```
# type "quit" to exit
Trying 2a00:1450:400c:c0d::6c...
Connected to smtp.gmail.com.
Escape character is '^]'.
220 smtp.gmail.com ESMTP ffacd0b85a97d-3df4fd372ccsm8640932f8f.32 - gsmtp
```
Follow logs live with:
```
journalctl -f -t postfix/smtpd -t postfix/smtp -t postfix/qmgr -t postfix/pickup -t postfix/cleanup
```
(send with `swaks` in a different window)

----

