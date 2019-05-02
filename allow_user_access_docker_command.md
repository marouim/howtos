# Allow user to access docker command

if docker group does not exists, create it. 

```
groupadd docker
```

Add the user to docker group

```
usermod -aG docker your_username
```

Exit the shell and login back
