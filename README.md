
# Kubernetes auth with Vault



## kubernetes secrets 
Secrets can be stored in kubernetes, files or data can be saved.

below is an example of logging into vault, and saving the login token as a secret in k8s.

```bash
$ vault login # create .vault-token in home dir.
$ kubectl create secret generic token --from-file=/home/lg/.vault-token
# secret/token created

$ kubectl get secrets
# NAME                  TYPE                                  DATA   AGE
# default-token-zplkr   kubernetes.io/service-account-token   3      23d
# token                 Opaque                                1      2m11s
$ kubectl describe secrets/token
# Name:         token
# Namespace:    default
# Labels:       <none>
# Annotations:  <none>
# 
# Type:  Opaque
# 
# Data
# ====
# .vault-token:  26 bytes
```

## vault policies 

Policies are HCL documents that apply policy to the vault paths and various engines granting access to crud actions.

policies are attched to tokens and app roles and grant authorization to resources.

## vault tokens

Tokens are used as an authentication method with vault.

Scoped tokens can be created with given policies attached to it.

### creating policies from hcl files

```bash
# load kv_role_gen.hcl and kv_app.hcl into vault
cat kv_role_gen.hcl | vault policy write kv_role_gen -
cat kv_app.hcl | vault policy write kv_app -
```
### creating a token with an attached policy

```bash
# create a token with policy
vault token create -policy=kv_role_gen
# Key                  Value
# ---                  -----
# token                s.IoIQg7rh3t3v0q0O0jvGA7Gw
# token_accessor       HOBu6oaYjAZCVAX7Ga1iM3aP
# token_duration       768h
# token_renewable      true
# token_policies       ["default" "kv_role_gen"]
# identity_policies    []
# policies             ["default" "kv_role_gen"]

```



## vault app roles

AppRoles have two elements a role id, that is public and a secret_id what is a private token. 

Vault policies are attached to give access to paths in vault.

### creating an approle and getting a secret_id
```bash
$ vault auth enable approle

# Create approle with attached policy kv_app

$ vault write auth/approle/role/kv_role policies=kv_app
# Success! Data written to: auth/approle/role/kv_role

# Read information on app role.

$ vault read auth/approle/role/kv_role
# WARNING! The following warnings were returned from Vault:
# 
#   * The "bound_cidr_list" parameter is deprecated and will be removed in favor
#   of "secret_id_bound_cidrs".
# 
# Key                      Value
# ---                      -----
# bind_secret_id           true
# bound_cidr_list          <nil>
# local_secret_ids         false
# period                   0s
# policies                 [kv_role_gen]

# secret_id_bound_cidrs    <nil>
# secret_id_num_uses       0
# secret_id_ttl            0s
# token_bound_cidrs        <nil>
# token_max_ttl            0s
# token_num_uses           0
# token_ttl                0s
# token_type               default

# Create a secret id

$ vault write -f auth/approle/role/kv_role/secret-id
# Key                   Value
# ---                   -----
# secret_id             2a6a8c86-0ffc-85bd-6d49-85928e2ef420
# secret_id_accessor    9ce12025-5f96-b62e-eb01-448c7bb2a360
#
```

## Brining it together

Now we have the ability to create a one time use vault token for the kv_role_gen policy, this will allow a one time creation for a secret ID.

This token is passed into a kubernetes secret, what can be attached to a pod to generate a secret id locally in memory to get a token for the kv_role approle with the attached kv_app policy.


```bash

$ vault read auth/approle/role/kv_role/role-id
# Key        Value
# ---        -----
# role_id    d433b865-d916-d0b7-e18e-82e68283774e


$ kubectl create secret generic kv-role-otp --from-literal=token=s.c0JFtfRJttX155cqoiF1UXPW --from-literal=role_id=d433b865-d916-d0b7-e18e-82e68283774e
# secret/kv-role-otp created


$ kubectl describe secret/kv-role-otp
# Name:         kv-role-otp
# Namespace:    default
# Labels:       <none>
# Annotations:  <none>
# 
# Type:  Opaque
# 
# Data
# ====
# token:    26 bytes
# role_id:  36 bytes


```

Giving kubernetes this one time use token, a pod can now be launched with this secret attached that can bootstrap the creation of a app_role secret_id that will only have touched the memory of the container. The following example could be ran in the container to generate the secret_id to complete the AppRole pairing.


```bash
vault write -f auth/approle/role/kv_role/secret-id
# Key                   Value
# ---                   -----
# secret_id             94d63410-2f7a-0e01-3b19-bb2878a3ccd9
# secret_id_accessor    42e65508-bf79-fc8f-d94d-b58f7e58b5f1

```

Now the container has a AuthN and AuthZ to the kv/* backend in vault, the secret_id has never crossed the network or left the memory of the container.

### Notes
hardening - encrypted swap