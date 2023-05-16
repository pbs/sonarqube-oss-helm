# Deploying SonarQube in K8 with external PostgreSQL backend

Refs:
    + https://github.com/SonarSource/helm-chart-sonarqube/tree/master/charts/sonarqube
    + https://gatesch.medium.com/deploying-sonarqube-in-k8-with-external-postgresql-backend-aca510712b37
    + https://medium.com/codex/easy-deploy-sonarqube-on-kubernetes-with-yaml-configuration-27f5adc8de90


## Initialize PostgreSQL Backend Database for SQ
```
-- Login to PostgreSQL
export POSTGRES_PASSWORD=postgres

kubectl run postgresql-dev-client --rm --tty -i --restart='Never' --namespace sonarqube --image postgres --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql-dev -U postgres -d postgres -p 5432

-- Create the user and database for sonarqube in postgresql
psql> CREATE USER sonarUser with password 'sonarpass';
psql> ALTER USER sonarUser WITH PASSWORD 'sonarpass';
psql> CREATE DATABASE sonardb with owner sonaruser encoding 'UTF8';
psql> \l   ## List Databases
psql> \c sonardb    ## Switch Database
psql> \dt
```

## SonarQube Installation
Deploy our SonarQube in Kubernetes backed by postgreSQL database.

### Taints & Tolerations - DO NOT USE THIS YET
Create a taint to avoid installing other pods 
```
kubectl get nodes
kubectl taint nodes esb-kube03-dev.pbs.org sonarqube=true:NoSchedule
```

### Get the SonarQube chart
```
cd charts/sonarqube
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update
helm dependency update
```
### Edit the “values.yaml” file 
1 - Enable the ingress and set the entry point URL
```
ingress:
enabled: true
# Used to create an Ingress record.
hosts:
-name: sonarqube.example.loc
```

2 - Set the CPU resource limit to 2000 millicores (it will considerably reduce the startup time for sonarQube)
```
resources:
limits:
cpu: 2000m
memory: 4096M
requests:
cpu: 400m
memory: 2Gi
```

3 - Increase the values for readiness, liveness and startup probes
```
readinessProbe:
initialDelaySeconds: 300
periodSeconds: 200
failureThreshold: 20

livenessProbe:
initialDelaySeconds: 300
periodSeconds: 200
failureThreshold: 20

startupProbe:
initialDelaySeconds: 120
periodSeconds: 60
failureThreshold: 240
```

4 - Set postgresql enabled to “false” since will be using an external postgreSQL, wneed to provide the postgreSQL server IP and credentials
```
postgresql:
# Enable to deploy the PostgreSQL chart
enabled: false

postgresqlServer: 192.168.12.41

postgresqlUsername: “sonaruser”
postgresqlPassword: “sonarpass”
postgresqlDatabase: “sonardb”
```

### Add new plugins
ref: https://github.com/mc1arke/sonarqube-community-branch-plugin

### Create the namespace for sonarqube in Kubernetes and deploy the helm chart
```
cd charts/sonarqube
kubectl create namespace sonarqube
helm upgrade --install -f pbs_values.yaml -n sonarqube sonarqube sonarqube/sonarqube \
    --set persistence.enabled=true,persistence.existingClaim=data-postgresql-dev-0

# or upgrade
helm upgrade -f pbs_values.yaml -n sonarqube sonarqube sonarqube/sonarqube \
    --set persistence.enabled=true,persistence.existingClaim=data-postgresql-dev-0

kubectl get all -n sonarqube

kubectl describe -n sonarqube pod/sonarqube-sonarqube-0

kubectl logs -n sonarqube pod/sonarqube-sonarqube-0

kubectl exec -n sonarqube sonarqube-sonarqube-0 -it -- //bin/bash
```
## Test Installation

```
export POSTGRES_PASSWORD=sonarpass
kubectl run postgresql-dev-client --rm --tty -i --restart='Never' --namespace sonarqube --image postgres --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql-dev -U sonaruser -d sonardb -p 5432

kubectl exec --stdin --tty --namespace sonarqube postgresql-dev-client -- psql --host postgresql-dev-0 -U sonaruser -d sonardb -p 5432

kubectl exec -n sonarqube pod/sonarqube-postgresql-0 -it -- //bin/bash
> psql -h localhost -U sonaruser --password -p 5432 sonardb

psql> \l   ## List Databases
psql> \dt
```

## Patch to NodePort
PS: new admin / P@ssw0rd
```
kubectl patch svc sonarqube-sonarqube -p '{"spec": {"type": "NodePort"}}' -n sonarqube
kubectl get all -n sonarqube
kubectl describe -n sonarqube pod/sonarqube-sonarqube-0
```

## Check Browser and Plugin

http://esb-kube03-dev.pbs.org:32593
http://esb-kube03-dev.pbs.org:32593/admin/marketplace?filter=installed


## Add SQ Plugin
Refs: 
    https://github.com/mc1arke/sonarqube-community-branch-plugin
    https://medium.com/cloudnesil/making-sonarqube-analysis-of-multiple-git-branches-with-docker-in-community-edition-48aaa768851



## Delete Installation
``` 
helm delete -n sonarqube sonarqube
kubectl -n sonarqube get all
kubectl delete namespace sonarqube
```

#

About this Repo
----------------

This is the Git repo of the SonarSource Helm Chart for [SonarQube](https://www.sonarqube.org/).  
The actual chart can be found in the [charts](charts/sonarqube) directory and see the README of the chart for more information. 

Have Question or Feedback?
--------------------------

For support questions ("How do I?", "I got this error, why?", ...), please first read the [documentation](https://docs.sonarqube.org) and then head to the [SonarSource Community](https://community.sonarsource.com/c/help/sq/10). The answer to your question has likely already been answered! 🤓

Be aware that this forum is a community, so the standard pleasantries ("Hi", "Thanks", ...) are expected. And if you don't get an answer to your thread, you should sit on your hands for at least three days before bumping it. Operators are not standing by. 😄


Contributing
------------

If you would like to see a new feature, please create a new Community thread: ["Suggest new features"](https://community.sonarsource.com/c/suggestions/features).

Please be aware that we are not actively looking for feature contributions. The truth is that it's extremely difficult for someone outside SonarSource to comply with our roadmap and expectations. Therefore, we typically only accept minor cosmetic changes and typo fixes.

With that in mind, if you would like to submit a code contribution, please create a pull request for this repository. Please explain your motives to contribute this change: what problem you are trying to fix, what improvement you are trying to make.

Willing to contribute to SonarSource products? We are looking for smart, passionate, and skilled people to help us build world-class code quality solutions. Have a look at our current [job offers here](https://www.sonarsource.com/company/jobs/)!

Note of Thanks
--------------

This chart was based on the great work done on the [Oteemo chart](https://github.com/Oteemo/charts/tree/master/charts/sonarqube). 
We would like to thank everyone who contributed for their great work on this project.

License
-------

Licensed under the [MIT Licence](LICENSE)
