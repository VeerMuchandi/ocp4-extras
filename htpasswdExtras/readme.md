# htpasswd - Changing Passwords or Adding Users

OpenShift docs in [Configuring an HTPasswd identity provider](https://docs.openshift.com/container-platform/4.2/authentication/identity_providers/configuring-htpasswd-identity-provider.html) explain how to create htpasswd file and how to add it as an authentication provider. But they don't explain how to add new users when you need to. Here the details:

## Add users or change passwords

```htpasswd <passwdfile> <username>```

example

```
% htpasswd myhtpasswd user2   
New password: 
Re-type new password: 
Updating password for user user2
```

## Updating OAuth in OpenShift

Now take the newly generated htpasswd file and create a secret in `openshift-config` project as shown below

```
% oc create secret generic htpasswd-secret --from-file=htpasswd=myhtpasswd -n openshift-config
secret/htpasswd-secret created
```

### Change the OAuth Custom Resource

Navigate to `Administration`->`Cluster Settings`->`Global Configuration`

Find the CR `OAuth`. Click on it and  navigate to `YAML` tab

Edit the spec to point to your newly created secret

```
..
..
..

spec:
  identityProviders:
    - htpasswd:
        fileData:
          name: htpasswd-secret
      mappingMethod: claim
      name: htpasswd
      type: HTPasswd
```

and save the CR.

This will trigger the operator to update the htpasswd file.