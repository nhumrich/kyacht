# KYacht
Kustom Yacht

KYacht is a kubernetes (k8s) templating library. It is very similar to kustomize/helm. 

## Purpose
The purpose of KYacht is to help ease the pain of verbose and many k8s files. 
KYacht will take a single service definition file, and apply all the values to a template. 

The templating engine uses jinja, the same template engine used in ansible/salt-stack. 

KYacht uses a single file to create many other components, allowing all components to share values if needed.

## FAQ

* Why not just use Helm/Kustomize/Other?
This library was built with frustration of using helm/kustomize. Helm has a pretty high barrier to entry, and personal preference felt that it was overly verbose. 
Kustomize is decent, but requires a *lot* of boilerplate and files to do simple things. 

Both of these two tools are designed to be "generic kubernetes templates" for anything in kubernetes. KYacht however, is designed specifically as a templating framework for services to be deployed. 

* Why is this library not written in Go?
Jinja is a python library. 
