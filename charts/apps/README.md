# App to create all my apps
The common chart for each deployed app from another repo.

The individual app specific charts live in the apps own repo. This parent helm repo simply defines them so they are deployed and synced automatically.


# Create the `ecr-registry-secret`
```
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=<ecr-registry> \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region eu-west-2) \
  --docker-email=<email>
```

# External secrets
https://external-secrets.io/latest/introduction/getting-started/

## Create AWS secrets credentials secret
```
echo -n 'KEYID' > ./access-key
echo -n 'SECRETKEY' > ./secret-access-key
kubectl create secret generic awssm-secret --from-file=./access-key --from-file=./secret-access-key
```
