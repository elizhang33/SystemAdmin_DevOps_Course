1. create and store key for BSD System, copy the keys to local machine.
2. create and store key for Ubuntu system, copy the keys to local machine.
3. Agent forwarding enabled in my local SSH client, keys are appropriately defined for both hosts.
4. here is the contents of my ~/.ssh/config file:
Host github.com
  AddKeysToAgent yes
  IdentityFile ~/.ssh/id_ecdsa

Host ada 
  Hostname linux.cs.pdx.edu
  IdentityFile ~/.ssh/squids/foo
  ForwardAgent yes
 
Host gitlab.com
	hostname cecs.pdx.edu
	IdentityFile ~/.ssh/id_ed25519_gitlab

![Screenshot](ifconfigAndhostname_screenshot.png)
![Screenshot](UbuntuOutput_screenshot.png) 

5. Sucessfully configured Ubuntu by following the instruction. 
