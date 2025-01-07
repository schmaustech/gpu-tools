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

The first step in this process is to create the `dockerfile.tools` file.
