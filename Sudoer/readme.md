
<h1>How to make a user sudoer?</h1>

Sometimes we may want users to be granted access to run certain commands as administrators. We can grant them cluster-admin privileges by running `oc adm policy add-cluster-role-to-user <username> cluster-admin`. However this should be avoided as much as possible.

Instead you can make the user a `sudoer` so that the user can run specific commands by impersonating as `system:admin` when needed. 

Adding `sudoer` role is easy.

<h2>Steps</h2>

1. Login as an administrator

2. Grant `sudoer` access

```
oc adm policy add-cluster-role-to-user sudoer <username> --as system:admin
```
Now the user with <username> can run any command as an administrator by adding `--as system:admin` at the end of the command

**Example:**

```
% oc get nodes
Error from server (Forbidden): nodes is forbidden: User "username" cannot list resource "nodes" in API group "" at the cluster scope

% oc get nodes --as system:admin
NAME                             STATUS   ROLES    AGE    VERSION
master0   Ready    master   191d   v1.14.6+8e46c0036
master1   Ready    master   191d   v1.14.6+8e46c0036
master2   Ready    master   191d   v1.14.6+8e46c0036
worker0  Ready    worker   191d   v1.14.6+7e13ab9a7
worker1   Ready    worker   191d   v1.14.6+7e13ab9a7
```




