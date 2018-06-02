####  # this howto is tensorflow/jupyter running in an openshift container
###### # follow these instructions to enable your node for GPU containers
##### #  
##### # now import the base tensorflow image
```
oc import-image repo.home.nicknach.net/tensorflow/tensorflow:latest-gpu -n openshift --insecure --confirm
```
##### # now run the template to create you project and prep it (with serviceaccount, scc, and device plugin daemonset)
```
oc create -f https://raw.githubusercontent.com/nnachefski/ocpstuff/master/ml/tensorflow.yml
```
##### # now use new-app to launch the image
```
oc new-app -i tensorflow:latest-gpu -e NVIDIA_VISIBLE_DEVICES=all -e NVIDIA_DRIVER_CAPABILITIES="compute,utility" -e NVIDIA_REQUIRE_CUDA="cuda>=8.0"
```
