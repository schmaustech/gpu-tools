# Build RDMA GPU-Tools Container

The purpose of this repository is to build a container that contains the testing tooling for validating RDMA connectivity and performance when used in conjunction with NVIDIA Network Operator and NVIDIA GPU Operator.  Specifically I want to be able to use the `ib_write_bw` command with the `--use_cuda` switch to demonstrate RDMA from one GPU in a node to another GPU in another node in an OpenShift cluster.  The `ib_write_bw` command is part of the perftest suite which is a collection of tests written over uverbs intended for use as a performance micro-benchmark. The tests may be used for HW or SW tuning as well as for functional testing.

The collection contains a set of bandwidth and latency benchmark such as:

* Send        - ib_send_bw and ib_send_lat
* RDMA Read   - ib_read_bw and ib_read_lat
* RDMA Write  - ib_write_bw and ib_write_lat
* RDMA Atomic - ib_atomic_bw and ib_atomic_lat
* Native Ethernet (when working with MOFED2) - raw_ethernet_bw, raw_ethernet_lat

In previous write ups I used a Fedora 35 container and manually added the components I wanted but here we will provide the tooling to build a container that will instantiate itself upon deployment of the container.  The workflow is as follows:

* Dockerfile.tools - which provides the content for the base image and the entrypoint.sh script.
* Entrypoint.sh - which provides the start up script for the container to pull in both the NVIDIA cuda libraries and also build and deploy the perftest suite with the cuda option available.
* Additional RPMs - there are some packages that were not part of the UBI image repo but are dependencies for CUDA toolkit.

The first thing we need to do is create a working directory for our files and an rpms directory for the rpms we will need for our base image.  I am using root here but it could be a regular user as well.

~~~bash
$ mkdir -p /root/gpu-tools/rpms
~~~

Next we need to download the following rpms from [Red Hat Package Downloads](https://access.redhat.com/downloads/content/package-browser) and place them into the rpms directory.

* infiniband-diags-51.0-1.el9.x86_64.rpm
* libglvnd-opengl-1.3.4-1.el9.x86_64.rpm
* libibumad-51.0-1.el9.x86_64.rpm
* librdmacm-51.0-1.el9.x86_64.rpm
* libxcb-1.13.1-9.el9.x86_64.rpm
* libxcb-devel-1.13.1-9.el9.x86_64.rpm
* libxkbcommon-1.0.3-4.el9.x86_64.rpm
* libxkbcommon-x11-1.0.3-4.el9.x86_64.rpm
* pciutils-devel-3.7.0-5.el9.x86_64.rpm
* rdma-core-devel-51.0-1.el9.x86_64.rpm
* xcb-util-0.4.0-19.el9.x86_64.rpm
* xcb-util-image-0.4.0-19.el9.x86_64.rpm
* xcb-util-keysyms-0.4.0-17.el9.x86_64.rpm
* xcb-util-renderutil-0.3.9-20.el9.x86_64.rpm
* xcb-util-wm-0.4.1-22.el9.x86_64.rpm

Once we have all our rpms for the base image we can move onto creating the `dockerfile.tools` file which we will use to build our image.

~~~bash
$ cat <<EOF >dockerfile.tools
# Start from UBI9 image
FROM registry.access.redhat.com/ubi9/ubi:latest

# Set work directory
WORKDIR /root
RUN mkdir /root/rpms
COPY ./rpms/*.rpm /root/rpms/

# DNF install packages either from repo or locally
RUN dnf install `ls -1 /root/rpms/*.rpm` -y
RUN dnf install wget procps-ng pciutils jq iputils ethtool net-tools git autoconf automake libtool -y

# Cleanup 
WORKDIR /root
RUN dnf clean all

# Run container entrypoint
COPY entrypoint.sh /root/entrypoint.sh
ENTRYPOINT ["/bin/bash", "/root/entrypoint.sh"]
EOF
~~~

We also need to create the `entrypoint.sh` script which is referenced in the dockerfile.

~~~bash
$ cat <<EOF > entrypoint.sh 
#!/bin/bash
# Set working dir
cd /root

# Configure and install cuda-toolkit
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
dnf clean all
dnf -y install cuda-toolkit-12-6

# Export CUDA library paths
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
export LIBRARY_PATH=/usr/local/cuda/lib64:$LIBRARY_PATH

# Git clone perftest repository
git clone https://github.com/linux-rdma/perftest.git

# Change into perftest directory
cd /root/perftest

# Build perftest with the cuda libraries included
./autogen.sh
./configure CUDA_H_PATH=/usr/local/cuda/include/cuda.h
make -j
make install

# Sleep container indefinitly
sleep infinity & wait
EOF
~~~

Next we can use the dockerfile we just created to build the base image.

~~~bash
$ podman build -f dockerfile.tools -t quay.io/redhat_emp1/ecosys-nvidia/gpu-tools:0.0.1
STEP 1/10: FROM registry.access.redhat.com/ubi9/ubi:latest
STEP 2/10: WORKDIR /root
--> Using cache 75f163f12503272b83e1137f7c1903520f84493ffe5aec0ef32ece722bd0d815
--> 75f163f12503
STEP 3/10: RUN mkdir /root/rpms
--> Using cache ade32aa6605847a8b3f5c8b68cfcb64854dc01eece34868faab46137a60f946c
--> ade32aa66058
STEP 4/10: COPY ./rpms/*.rpm /root/rpms/
--> Using cache 59dcef81d6675f44d22900f13a3e5441f5073555d7d2faa0b2f261f32e4ba6cd
--> 59dcef81d667
STEP 5/10: RUN dnf install `ls -1 /root/rpms/*.rpm` -y
--> Using cache ebb8b3150056240378ac36f7aa41d7f13b13308e9353513f26a8d3d70e618e3b
--> ebb8b3150056
STEP 6/10: RUN dnf install wget procps-ng pciutils jq iputils ethtool net-tools git autoconf automake libtool -y
--> Using cache 5ca85080c103ba559994906ada0417102f54f22c182bbc3a06913109855278cc
--> 5ca85080c103
STEP 7/10: WORKDIR /root
--> Using cache 68c8cd47a41bc364a0da5790c90f9aee5f8a8c7807732f3a5138bff795834fc1
--> 68c8cd47a41b
STEP 8/10: RUN dnf clean all
Updating Subscription Management repositories.
Unable to read consumer identity

This system is not registered with an entitlement server. You can use subscription-manager to register.

26 files removed
--> a219fec5df49
STEP 9/10: COPY entrypoint.sh /root/entrypoint.sh
--> aeb03bf74673
STEP 10/10: ENTRYPOINT ["/bin/bash", "/root/entrypoint.sh"]
COMMIT quay.io/redhat_emp1/ecosys-nvidia/gpu-tools:0.0.1
--> 45c2113e5082
Successfully tagged quay.io/redhat_emp1/ecosys-nvidia/gpu-tools:0.0.1
45c2113e5082fb2f548b9e1b16c17524184c4079e2db77399519cf29829af1e7
~~~

Once the image is created we can push it to our favorite registry.

~~~bash
$ podman push quay.io/redhat_emp1/ecosys-nvidia/gpu-tools:0.0.1
Getting image source signatures
Copying blob 62ee1c6c02d5 done   | 
Copying blob 6027214db22e done   | 
Copying blob 4822ebd5a418 done   | 
Copying blob 422a0e40f90b done   | 
Copying blob 5916e2b21ab2 done   | 
Copying blob 10bf375a4d78 done   | 
Copying blob ca1c18e183d5 done   | 
Copying config 3bbb6e1f9b done   | 
Writing manifest to image destination
~~~

Now that we have an image let's test it out on the system where we have compatible RDMA hardware configured.  I am using the same setup as I used in a previous write up so I am going to skip the details about setting up a service account and providing the privileges to it.   We will however create our workload pod yaml which we will use to deploy the image.

~~~bash
cat >>EOF >rdma-32-workload.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-eth-32-workload
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: rdmashared-net
spec:
  nodeSelector: 
    kubernetes.io/hostname: nvd-srv-32.nvidia.eng.rdu2.dc.redhat.com
  serviceAccountName: rdma
  containers:
  - image: quay.io/redhat_emp1/ecosys-nvidia/gpu-tools:0.0.1
    name: rdma-32-workload
    command:
      - sh
      - -c
      - sleep inf
    securityContext:
      privileged: true
      capabilities:
        add: [ "IPC_LOCK" ]
    resources:
      limits:
        nvidia.com/gpu: 1
        rdma/rdma_shared_device_eth: 1
      requests:
        nvidia.com/gpu: 1
        rdma/rdma_shared_device_eth: 1
EOF
~~~

Next we can deploy the container.
