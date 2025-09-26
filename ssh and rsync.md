See site-mgmt/create.md for details about how to set up SSH keys

### rsync between servers

<details><summary>rsync setup</summary>

---
It is necessary to use rsync to get the most recent source folder when creating a new account

```
rsync -vaPur --delete /Users/Main/sourceserver/home/basen/SYNC/* /Users/Main/destserver/opt/SYNCEN/
```
generate SSH key on destination computer [link](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server) 

The first step to configure SSH key authentication to your server is to generate an SSH key pair on your local computer.

On your local computer, generate a SSH key pair by typing:
```
ssh-keygen
```
I did that, and overwrote the old key (which is good because it had been copied — I should add this to server creation)

#### Step 2 — Copying an SSH Public Key to the Distant Server

Copying Your Public Key Using ssh-copy-id, easiest
```
ssh-copy-id root@live.svija.com
```
SSH to live server to check if it worked (worked like a charm).

---
</details>

rsync to get current SYNC folder
```
rsync -vaPur --delete live.svija.com:/home/basefr/SYNC/ /opt/SYNCFR/
```
```
rsync -vaPur --delete live.svija.com:/home/baseen/SYNC/ /opt/SYNCEN/
```

---

