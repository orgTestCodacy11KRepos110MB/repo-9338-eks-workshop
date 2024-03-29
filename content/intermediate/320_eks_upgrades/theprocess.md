---
title: "The Upgrade Process"
weight: 321
---

The process goes as follows:

1. (Optional) Check if the new version you are upgrading to has any API deprecations which will mean that you'll need to change your YAML Spec files for them to continue to work on the new cluster. This is only the case with some version upgrades such as [1.21 to 1.22](https://kubernetes.io/blog/2019/07/18/api-deprecations-in-1-16/). There are various tools that can help with this such as [kube-no-trouble](https://github.com/doitintl/kube-no-trouble). Since there are not any such deprecations going from 1.21 to 1.22 we'll skip this step here.
1. Run a `kubectl get nodes` and ensure that all of your Nodes are running the current version. Kubernetes can only support nodes that are one version behind - meaning they all need to match the current Kubernetes version so when you upgrade the EKS control plane they'll then only be one version behind. For Fargate relaunching a Pod (maybe by deleted it and letting the ReplicaSet replace it) will bring it in line with the current Kubernetes version.
1. Upgrade the EKS Control Plane to the new major version
1. Check if the core add-ons (kubeproxy, CoreDNS and the CNI) that ship with EKS require an upgrade to coincide with the major version upgrade. This will be in our [upgrade documentation](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html#w665aac14c15b5c17). If so follow that documentation to upgrade those. In this case (a 1.21 to 1.22 upgrade) the documentation says we'll need to upgrade both CoreDNS and kubeproxy.
1. Upgrade any worker nodes so that the kubelet on them (which will now be one Kubernete major version behind) matches that of the EKS control plane. While you don't have to do this immediatly it is a good idea to have the Nodes on the same version as the control plane as soon as is practical - plus it will make Step 2 easier the next time you have to upgrade. If you are using Managed Node Groups (as we are here) then EKS can help facilitate this with a safe automated process which orchestrates both the AWS and Kubernetes side of a rolling Node replacement/upgrade. If you are using Fargate then this will happen automatically the next time your Pods are replaced.
