## This doc describes how setup content mirrors for disconnected Openshift installations (single or multi-master).
#### # In environments where you do not have access to the public internet (to perform the install), you'll need to make a local copy of rpms and container images.  You would then configure the Openshift installation to pull it's content from this repo.
###### # You could also use a Red Hat Satellite for this purpose.  This requires additional resources and time to configure the satellite server (which may be out-of-scope for a quick Openshift PoC).
###### # If you are interested in setting up RH Satellite for this content mirror, the instruction for that are here:
###### # https://github.com/nnachefski/ocpstuff/tree/master/satellite    

##### # You will need some local storage on the repo server (called 'repo.home.nicknach.net' in this example) 
###### # recommended 200GB storage

##### # set your repo host vars
```
export MY_REPO=repo.home.nicknach.net
export SRC_REPO=registry.access.redhat.com
export OCP_VER=v3.11
export POOLID=8a85f98260c27fc50160c323263339ff
export RHN_ID=nnachefs@redhat.com
export RHN_PWD=
export STAGE_DIR=/var/deleteme #this will hold scripts and rpms before moved to /var/www/html
```
##### # or, if doing an internal puddle build
```
#export SRC_REPO=brew-pulp-docker01.web.prod.ext.phx2.redhat.com:8888
```
##### # subscribe the repo box to the proper channels for OCP
```
subscription-manager register --username=$RHN_ID --password $RHN_PWD --force
subscription-manager attach --pool=$POOLID
subscription-manager repos --disable="*"
subscription-manager repos \
   --enable=rhel-7-server-rpms \
   --enable=rhel-7-server-extras-rpms \
   --enable=rhel-7-server-ose-3.11-rpms \
   --enable=rhel-7-fast-datapath-rpms \
   --enable=rhel-7-server-ansible-2.6-rpms \
   --enable=rh-gluster-3-client-for-rhel-7-server-rpms \
   --enable=rhel-7-server-optional-rpms \
   --enable=rhel-server-rhscl-7-rpms

yum -y install yum-utils @development rh-python36
 
yum -y install rh-python36-numpy \
 rh-python36-scipy \ 
 rh-python36-python-tools \
 rh-python36-python-six
 
# exit
```
##### # open the firewall up
###### # you can get more strict with this if you want
```
firewall-cmd --set-default-zone trusted
```
##### # if doing puddle builds, you'll need to connect to the RH vpn first.
###### # you'll need these packages (look on mojo)
```
# redhat-internal-NetworkManager-openvpn-profiles-0.1-30.el7.csb.noarch.rpm
# redhat-internal-NetworkManager-openvpn-profiles-non-gnome-0.1-30.el7.csb.noarch.rpm
# redhat-internal-openvpn-profiles-0.1-30.el7.csb.noarch.rpm
```
##### # now connect with your two-factor (puddle builds only) 
```
#openvpn --config /etc/openvpn/ovpn-phx2-udp.conf
```
###### # 'ctrl+z' then 'bg'

### # create the docker image mirror on our repo server
##### # install/enable/start docker-distribution on the repo box
```
yum -y install docker-distribution.x86_64
```
##### # create certs for this registry (so you can enable https, required for v2 images)
```
mkdir -p /etc/docker/certs.d/$MY_REPO
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout /etc/docker/certs.d/$MY_REPO/$MY_REPO.key -x509 -days 365 -out /etc/docker/certs.d/$MY_REPO/$MY_REPO.cert
```
##### # add this to the http section in /etc/docker-distribution/registry/config.yml
```
cat <<EOF >> /etc/docker-distribution/registry/config.yml
    headers:
        X-Content-Type-Options: [nosniff]
    tls:
        certificate: /etc/docker/certs.d/$MY_REPO/$MY_REPO.cert
        key: /etc/docker/certs.d/$MY_REPO/$MY_REPO.key
EOF
```
##### # change the port from 5000 to 443
```
sed -i 's/\:5000/\:443/' /etc/docker-distribution/registry/config.yml
```
##### # restart registry
```
systemctl restart docker-distribution
```
##### # copy the cert to the webroot for your clients to pull from
```
cp -f /etc/docker/certs.d/$MY_REPO/$MY_REPO.cert /var/www/html/repo
```
##### # copy cert to local pki store and update
```
cp -f /etc/docker/certs.d/$MY_REPO/$MY_REPO.cert /etc/pki/ca-trust/source/anchors/$MY_REPO.cert && restorecon /etc/pki/ca-trust/source/anchors/$MY_REPO.cert && update-ca-trust
```
##### # setup the epel repo so we can get python36 package, then disable it
```
# Figure out rhscl before removing this
#rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
#yum install -y python36 python36-pip
#yum-config-manager --disable epel
```
##### # make sure skopeo is installed
```
yum install -y skopeo
```
##### # get the import-images.py script and image lists (this script is a python3 wrapper for skopeo)
```
mkdir -p $STAGE_DIR && cd $STAGE_DIR
wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/scripts/import-images.py && chmod +x import-images.py
wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/images/core_images.txt
wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/images/app_images.txt
wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/images/mw_images.txt  
```
##### # now copy the images to your repo (./import-images.py --help)
``` 
scl enable rh-python36 bash
for i in core_images.txt app_images.txt mw_images.txt; do
  ./import-images.py docker $SRC_REPO $MY_REPO -d -l $i
done
```
###### # you can add a tag alias like this, '-a v4.0.0'  (the registry would host both v4.0 & v4.0.0 tagged images)
##### # install/enable/start httpd
```
yum -y install httpd && systemctl enable httpd --now
```
##### # start the reposync
```
cd $STAGE_DIR && mkdir repo && wget https://raw.githubusercontent.com/nnachefski/ocpstuff/master/scripts/sync_repos.sh && chmod +x sync_repos.sh
./sync_repos.sh
```
##### # move and fix repo dir selinux
```
mv repo /var/www/html && restorecon -R /var/www/html
```
#### # Troubleshooting disconnected installs
##### # during the install, do these commands in separate terminals to trouble shoot any missing images
```
watch oc get pods -owide --all-namespaces

# and

watch oc get pv

# and

journalctl -xlf
```
###### # look for errors about images not found
##### # after install, you may find that you didnt copy EVERY single non-core image.  If that is the case, *some* imageStreams may reference non-existing (older) images.  If you want a list of missing (defunct) images, then do this
```
for i in `oc get is -n openshift |grep -v NAME |awk '{print $1}'`; do oc get is $i -n openshift -o json; done |grep 'not found' |awk '{print $3}' |awk -F \/ '{print $2,$3}' | awk -F \: '{print $1}' |sed 's/ /\//g' |sort -u
```
###### # this will yield a list of missing images that you can import using the import-images.py script
###### # redirect the output to a file (> missing.txt) and then re-run the import-images.py script with '-l missing.txt'
###### # (be prepared to pay a storage cost for all those old images)
