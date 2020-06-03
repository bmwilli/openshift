# Hello-node

The kubernetes example for minikube does not run as is with openshift.

This is because the container that it uses needs to execute as the root user.

In these steps we create a service account in openshift and assign it the SCC (Security Context Constraint) of anyuid

## Prerequisites
* Access to an admin account in openshift that can assign the SCC anyuid to a service account

## Steps


