# ssh-copy-id not working 

## check point

* home directory permission : 0755

  ```shell
  chmod 0755 /home/<username>
  ```

  > make sure your home directory has the right owner and group. e.g. 
  >
  > ```shell
  > $ ll / 
  > drwxr-xr-x.  20 centos centos 4096 Aug 21 15:29 root
  > #change to right owner and group
  > $ chown root:root /root
  > $ ll /
  > drwxr-xr-x.  20 root root 4096 Aug 21 15:29 root
  > ```
  >
  >

* .ssh/ directory permission : 700

  ```shell
  chmod 700 /home/<username>/.ssh
  ```

* .ssh/authorized_keys : 600

  ```shell
  chmod 600 /home/<username>/.ssh/authorized_keys
  ```

* check in /etc/ssh/sshd_config  make sure the lines are not commented

  ```shell
  RSAAuthentication yes
  PubkeyAuthentication yes
  AuthorizedKeysFile  .ssh/authorized_keys
  ```


## Ref

<https://superuser.com/questions/189376/ssh-copy-id-does-not-work>

<http://yangcongchufang.com/ssh-copy-id-not-work.html>