# RBAC Hands On

The exercise will be done on cluster available at ip address `169.61.143.190`

1. Login into the ICp console as admin.

2. Create a new namespace.

   In the following steps, the namespace you created will be referenced as `<your_namespace>`.

3. Create a team, <your_team>.

4. Import user1 as `viewer` of the team.

5. Associate your namespace `<your_namespace>` as resource of the team you created.

6. Configure the kubectl cli by using the information provided by console.

7. Create a file 'dev_role.yaml'.

8. Fill the file with the defition of new role who has the permission to manipulate docker images registry.

The file should be similar to :
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: icp:develop
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
rules:
- apiGroups: ["icp.ibm.com"]
  resources: ["images"]
  verbs: ["create", "get", "list", "patch", "update", "watch"]
```

9. Create the role and associate it your namespace `<your_namespace>`.

```
    kubectl create -f dev_role.yaml -n <your_namespace>
```

10. Do rolebinding between the new role and the user `user1` in the namespace `<your_namespace>`.

```
  kubectl create rolebinding icp:<your_team>:developer --role=icp:develop --user=https://mycluster.icp:9443/oidc/endpoint/OP#user1 --namespace=<your_namespace>
```

  the user `user1` is now able to push image into the namespace `<your_namespace>`.

11. Describe the role you created

```
  kubectl describe roles icp:develop -n <your_namespace>
```

11. Describe the binding you created

```
  kubectl describe rolebinding icp:<your_team>:developer -n <your_namespace>
```

11. login as user1 (id:user1, pwd: ChangeMe)
```
bx pr login -a https://169.61.143.190:8443 --skip-sll-validation
```

12. Check that user1 has no permission to create images on default your_namespace.

```
  kubectl auth can-i create images.icp.ibm.com
```

13. Check that user1 has  permission to create images on your namespace <your namespace>

```
  kubectl auth can-i create images.icp.ibm.com -n <your_namespace>
```
