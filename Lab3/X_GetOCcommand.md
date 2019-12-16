
# OC Command Install

Download Site    
[oc command](https://mirror.openshift.com/pub/openshift-v4/clients/oc/4.1/)

``` 
$ cd ~/
$ mkdir -v ./src && cd ./src/
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/oc/4.1/linux/oc.tar.gz
$ tar zxvf ./oc.tar.gz
$ sudo cp -vi ./oc /usr/local/bin/
$ oc --help
```

## OC Command からのログイン

```
$ oc login <OpenShift Portal>
```
