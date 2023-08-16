# **Neo4j deployment in K8s**

## **Requirements**

- EBS CSI DRIVER `Will be added in IaC` [@OfficialDocumentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)



### **Kustomization**

We're using kustomization to deploy all the ressources, in the main root there's `kustomization.yaml` that specifies every deployment/service/pvc folder and also using it for secrets/configmap.


**Why too many kustomization file**

We have a file in the root called kustomization.yaml and it specifies the folder that has file manifests.
Let's say we have 5 folder and each of them has X file of manifests, to apply all of them we'll have to `cd` to each folder and use `kustomize build . | kubectl apply -f -` so instead of this, we'll create a folder in the root that calls all the kustomizations in each folder and we won't need to `cd` to all of them. we'll just execute it in the root and all the manifests will be applied

---

Add them to your repository's secrets : 

- Go to your repository's settings.

- Under the settings, navigate to the 'Secrets' section.

- Click on 'New repository secret' or 'Add a secret environment' button.

- For the name, enter `NEO4J_AUTH`.

- Input the new master key value in the 'Value' field.

:warning: You must put `username/password` in the secret for `NEO4J_AUTH`, example : `neo4j/thisiscomplicated`

A file named secret.env will be created and will contain all the secrets created in the github's repository

```
      - name: Create Secret file
        env:
          SECRETS: '${{ toJson(secrets) }}'
        run: |
          for secret in $(echo "$SECRETS" | jq -r 'keys[] | select(. != "github_token")'); do
            value="$(jq -r --arg key "$secret" '.[$key]' <<< "$SECRETS")"
            echo "$secret=$value" >> secret.env
          done

```
It will create a secret based on these variables and can be modified only in Github repo's secret variables.


You can also edit your configmap by editing `configmap.env`.

```
  NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
  NEO4J_EDITION=ENTERPRISE_K8S
  NEO4J_CONF=/config/
  K8S_NEO4J_NAME=neo4j
  EXTENDED_CONF=yes
```

In the workflow, a step will read this .env and generate configmap.yaml from it

---

### Neo4j

In this documentation, we're using only 1 server and 1 replica since we're using a `persistentVolumeClaim` in `statefulset.yaml`.

to be able to deploy more replicas, we will have to replace the `persistentVolumeClaim` by a `volumeClaimTemplates` and this will create a new PersistentVolumeClaim (and a new PersistentVolume) for each replica. Data is not shared, except to the extent the container process knows how to replicate data between its replicas. If a StatefulSet Pod is deleted and recreated, it will come back with the same underlying PVC and the same data, even if it is recreated on a different Node.

```
  volumeClaimTemplates: 
    - metadata:
        name: "data"
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 30Gi
        storageClassName: gp3-retain-sc
```

Once everything is setup, To apply all the resources, run the following command:

`kustomize build . | kubectl apply -f - `

If you want to delete all the resources:

`kustomize build . | kubectl delete -f - `


### Testing 

We can connect to our neo4j by using 

- cypher-shell = ` cypher-shell -u neo4j -p "yourpassword"`

  You can show all your databases by typing `SHOW DATABASES`
  You can show all your servers by typing `SHOW SERVERS`

  You can exit by typing `:exit;`

- browser = `kubectl port-forward -n default service/app-neo4j 7474:7474 7687:7687`

In connect URL = `bolt://localhost:7687` or if you're using ingress, `bolt+s://domain.com:443`
Username & Password = `neo4j` & `yourpassword`

----------------------------------------------------------------------------------------------------------------------------

For the ingress, you must install ALB : [@ALBIngress](https://github.com/TrouveTaVoie/eks-cluster)

Request your domain in AWS ACM. Once the request is approved and the certificate is issued, copy the Certificate ARN in `./openfga/ingress.yaml`

File : `./openfga/ingress.yaml` (Line 10)
```
    alb.ingress.kubernetes.io/certificate-arn: <yourcertificate> # Certificate ARN in ACM (for your domain)
```

Next, change the domain to your desired domain:

File : `ingress.yaml` (Line 16)
```
    - host: "your.domain.com"
```
