# Shell Commands

1 synchronize clocks with time server

```shell
ntpdate time.ntp.org
```

2 `find` 

```shell
#find all directories and delete them, only keep files，  DANGEROUS!
find ./ -maxdepth 1 -type d -not -path "./" | xargs -i rm -rf {}

#find all ".go" files that contain "kuard"
find ./ -name "*.go" -exec grep -in "kuard" {} \; 
find ./ -name "*.go" | xargs  grep -in "kuard"  

#find all ".go" or ".sh" files that contain "kuard"
find ./ \( -name "*.go" -or -name "*.sh" \) -exec grep -in "kuard" {} \;
```

3 `sed`

```shell
# find all files that contain "old-string" and replace them to "new-string-to-replace"
find ./ -type f | xargs -i sed -i 's/old-string/new-string-to-replace/g'  {}
sed -i "s/old-string/new-string-to-replace/g"  `grep  "old-string" -rl ./`

#delete blank lines or lines consisting of spaces
sed -i '/^[ ]*$/d' images.txt

```

4 `scp`

```shell
#send file to remote by a jump server using scp with ProxyCommand subcommand
scp -o "ProxyCommand ssh <user>@<Proxy-Server> nc %h %p" <File-Name> User@<Destination-Server:<Destination-Path>
```

5 `ansible`

```shell
#using ansible to delete all lines that contain "match-string" in remote server 's file  /etc/hosts 
#host_file is a file that list all remote server ips
ansible -i host_file all -m shell -a "sed -i /match-string/d /etc/hosts"
```

6 `docker`

```shell
#load all tar images files 
ls -1 *.tar | xargs --no-run-if-empty -L 1 dockerload -i

#stop all running docker and remove them
docker ps  | awk ' NR>1 {print $1}' | xargs -i docker stop {} && docker rm {}
```

7 `for` loop

```shell
for f in $(cat images.txt ) ; do wget $f; done 
```

8 `yum` install `netstat`

```shell
yum install net-tools
```

9 `kubectl`

#find mapping nodes-pvc-pv
kubectl get pvc -o go-template --template='{{range .items}}{{index .metadata.annotations "volume.kubernetes.io/selected-node"}} {{"-"}} {{index .metadata.name}} {{" - "}} {{index .spec.volumeName}}  {{"\n"}}{{end}}'

```
#get pods uid and name mapping
kubectl get pods -n jd-tpaas -o json |jq ".items[] | {name: .metadata.name, uid: .metadata.uid}" -r
```



