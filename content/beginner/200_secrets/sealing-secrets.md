---
title: "Sealing Your Secrets"
date: 2021-07-15T00:00:00-03:00
weight: 14
draft: false
---

First, let's delete the **database-credentials** Secret and Pod resources that was created earlier in this module and deployed to the *octank* namespace in the cluster. After this operation, the only Secret that should exist in that namespace will be that of the token generated by Kubernetes for the default service account associated with the *octank* namespace.

```
kubectl delete pod pod-variable pod-volume -n octank 
kubectl delete secret database-credentials -n octank
kubectl get secret -n octank
```
Output:
{{< output >}}
NAME                  TYPE                                  DATA   AGE
default-token-gv8nr   kubernetes.io/service-account-token   3      2d23h
{{< /output >}}

Now, let's reuse the Secret created previously to create SealedSecret YAML manifests with *kubeseal*.
```
cd ~/environment/secrets
kubeseal --format=yaml < secret.yaml > sealed-secret.yaml
```

An alternative approach is to fetch the public key from the controller and use it offline to seal your Secrets
```
kubeseal --fetch-cert > public-key-cert.pem
kubeseal --cert=public-key-cert.pem --format=yaml < secret.yaml > sealed-secret.yaml
```

View the contents of the regular Secret and the corresponding SealedSecret with the following commands:
```
cat secret.yaml 
cat sealed-secret.yaml 
```
Output of secret.yaml :
{{< output >}}
apiVersion: v1
data:
  password: VHJ1NXROMCE=
  username: YWRtaW4=
kind: Secret
metadata:
  name: database-credentials
  namespace: octank
type: Opaque
{{< /output >}}

Output of sealed-secret.yaml:
{{< output >}}
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: database-credentials
  namespace: octank
spec:
  encryptedData:
    password: AgBR2S0Orv6YKLgEboBbPBsV6PX9gP9gUUTrISv(...)daQtadX6XnFqoYwOQ==
    username: AgBtkiM4Q7w3kesVDbZgKOpiVQ+9XMmlqr9xABf(...)B9quqLBF80QFSsZpQ==
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: database-credentials
      namespace: octank
    type: Opaque
{{< /output >}}

Note that the keys in the original Secret, namely, *username* and *password*, are not encrypted in the SealedSecret; only their values. You may change the names of these keys, if necessary, in the SealedSecret YAML file and still be able to deploy it successfully to the cluster. However, you cannot change the name and namespace of the SealedSecret. The SealedSecret and Secret **must have the same namespace and name**.

Now, deploy the SealedSecret to your cluster:
```
kubectl apply -f sealed-secret.yaml 
```
Looking at the logs of the contoller, you can see that it picks up the SealedSecret custom resource that was just deployed, unseals it to create a regular Secret.
```
kubectl logs -n kube-system sealed-secrets-controller-7bdbc75d47-5wxvf
```
Output:
{{< output >}}
(...)
2021/07/14 21:27:01 HTTP server serving on :8080
2021/07/15 13:22:20 Updating octank/database-credentials
2021/07/15 13:22:20 Event(v1.ObjectReference{Kind:"SealedSecret", Namespace:"octank", Name:"database-credentials", UID:"abc9b6ab-fe69-453a-8654-c9593de935c7", APIVersion:"bitnami.com/v1alpha1", ResourceVersion:"104915", FieldPath:""}): type: 'Normal' reason: 'Unsealed' SealedSecret unsealed successfully
{{< /output >}}

Verfiy that the **database-credentials** Secret unsealed from the SealedSecret was deployed by the controller to the *octank* namespace.
```
kubectl get secret -n octank database-credentials
```
Output:
{{< output >}}
NAME                   TYPE     DATA   AGE
database-credentials   Opaque   2      90s
{{< /output >}}

Redeploy the pod that reads from the above Secret and verify that the keys have been exposed as environment variables with the correct literal values.
```
kubectl apply -f pod-variable.yaml
kubectl wait -n octank pod/pod-variable --for=condition=Ready
kubectl logs -n octank pod-variable
```
Output:
{{< output >}}
pod/pod-variable created
pod/pod-variable condition met
DATABASE_USER = admin
DATABASE_PASSWROD = Tru5tN0!
{{< /output >}}

The YAML file, **sealed-secret.yaml**, that pertains to the SealedSecret is safe to be stored in a Git repository along with YAML manifests pertaining to other Kubernetes resources such as DaemonSets, Deployments, ConfigMaps etc. deployed in the cluster. You can then use a [GitOps workflow](https://www.weave.works/technologies/gitops/) to manage the deployment of these resources to your cluster. The YAML file, **secret.yaml**, that pertains to the Secret may be deleted because it is never used in any subsequent workflows.
