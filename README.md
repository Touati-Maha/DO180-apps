# Configuration de Loki et OpenShift Logging

Ce guide explique comment configurer Loki pour la collecte et le stockage des logs avec OpenShift Logging. Deux options sont disponibles :  
1. Utiliser un backend S3 externe (AWS S3, MinIO, ou autre).  
2. Utiliser OpenShift Data Foundation (ODF) pour le stockage S3 interne.

---

## Prérequis

- Un cluster OpenShift configuré: Red Hat OpenShift Logging operator & Loki operator sont installés
- Soit un service S3 externe compatible, soit OpenShift Data Foundation (ODF) installé.
- Pour un backend S3 externe :  
  - **Access Key**, **Secret Key**, **Bucket Name** et **Endpoint URL**.


---

## Option 1 : Utiliser un backend S3 externe

### Étape 1.1 : Créer le Secret pour S3 externe

Créez un secret contenant vos identifiants S3 et informations de connexion :

```
oc create secret generic logging-loki-s3 -n openshift-logging \
  --from-literal=access_key_id=<ACCESS_KEY> \
  --from-literal=access_key_secret=<SECRET_KEY> \
  --from-literal=bucketnames=<BUCKET_NAME> \
  --from-literal=endpoint=<S3_ENDPOINT_URL>
```
Remplacez :

<ACCESS_KEY> : Clé d'accès S3.
<SECRET_KEY> : Clé secrète S3.
<BUCKET_NAME> : Nom du bucket.
<S3_ENDPOINT_URL> : URL d'accès au service S3 (par exemple s3.amazonaws.com ou l'URL MinIO).

### Étape 1.2 : Configurer LokiStack pour S3 externe
Créez un fichier lokistack-s3.yaml :

```
Copier le code
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.demo
  storage:
    secret:
      name: logging-loki-s3
      type: s3
  tenants:
    mode: openshift-logging
```
Appliquez cette configuration :

```
oc apply -f lokistack-s3.yaml
```
##  Option 2 : Utiliser OpenShift Data Foundation (ODF)

### Étape 2.1 : Créer une revendication de bucket

Créez un fichier loki-bucket-claim.yaml :

```
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: loki-bucket-odf
  namespace: openshift-logging
spec:
  generateBucketName: loki-bucket-odf
  storageClassName: openshift-storage.noobaa.io
Appliquez cette configuration :
```

```
oc apply -f loki-bucket-claim.yaml
```

###  Étape 2.2 : Créer un Secret basé sur le bucket ODF
Une fois le bucket créé, exécutez les commandes suivantes pour générer un secret :

```
oc create secret generic logging-loki-odf -n openshift-logging \
  --from-literal=access_key_id=$(oc get -n openshift-logging secret loki-bucket-odf -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d) \
  --from-literal=access_key_secret=$(oc get -n openshift-logging secret loki-bucket-odf -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d) \
  --from-literal=bucketnames=$(oc get -n openshift-logging configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_NAME}') \
  --from-literal=endpoint=https://$(oc get -n openshift-logging configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_HOST}'):$(oc get -n openshift-logging configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_PORT}')
```

###  Étape 2.3 : Configurer LokiStack pour ODF
Créez un fichier lokistack-odf.yaml :

```
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.demo
  storage:
    secret:
      name: logging-loki-odf
      type: s3
    tls:
      caName: openshift-service-ca.crt
  tenants:
    mode: openshift-logging
Appliquez cette configuration :
```
```
oc apply -f lokistack-odf.yaml
```

##  Étapes communes pour OpenShift Logging

###  Étape 3 : Configurer ClusterLogging

Créez un fichier clusterlogging.yaml :

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  collection:
    type: vector
  logStore:
    lokistack:
      name: logging-loki
    type: lokistack
  visualization:
    type: ocp-console
  managementState: Managed
```
Appliquez cette configuration :

```
oc apply -f clusterlogging.yaml
```

###  Étape 4 : Configurer ClusterLogForwarder
Créez un fichier clusterlogforwarder.yaml :

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  pipelines:
  - name: all-to-default
    inputRefs:
    - infrastructure
    - application
    - audit
    outputRefs:
    - default
```

Appliquez cette configuration :

```
oc apply -f clusterlogforwarder.yaml
```
##  Vérifications
Vérifiez que tous les pods dans le namespace openshift-logging sont en état Running :

```
oc get pods -n openshift-logging
```

```
NAME                                           READY   STATUS    RESTARTS   AGE
cluster-logging-operator-554849f7dd-9tcz2      1/1     Running   0          24m
collector-7rb2n                                1/1     Running   0          93s
collector-bkj8x                                1/1     Running   0          93s
collector-cng5z                                1/1     Running   0          93s
collector-hx2gd                                1/1     Running   0          93s
collector-x92zq                                1/1     Running   0          93s
collector-xlqw9                                1/1     Running   0          93s
logging-loki-compactor-0                       1/1     Running   0          98s
logging-loki-distributor-64c798c4c5-6wpxp      1/1     Running   0          99s
logging-loki-gateway-68fb59cdf5-6b8mt          2/2     Running   0          98s
logging-loki-gateway-68fb59cdf5-7qf4s          2/2     Running   0          98s
logging-loki-index-gateway-0                   1/1     Running   0          98s
logging-loki-ingester-0                        1/1     Running   0          99s
logging-loki-querier-577b55f8d5-f4cfb          1/1     Running   0          98s
logging-loki-query-frontend-775755684d-f94bd   1/1     Running   0          98s
logging-view-plugin-5b9b5b7bdc-tvkqk           1/1     Running   0          94s
```
Une fois que vous avez activé la console Web pour la visualisation, vous pouvez afficher vos journaux dans le menu Observe → Logs. L’image suivante montre le menu Logs et les filtres que vous pouvez appliquer à vos journaux à l’aide de la console Web :

![Logs ](webui.png)

1- Plage de temps des journaux.

2- Filtrage des journaux par contenu, namespace, pod ou conteneur.

3- Niveau de gravité du journal. Les valeurs possibles sont critical, error, warning, debug, info, trace et unknown.

4- Type de journal. Les valeurs possibles sont application, infrastructure et audit.

5- Requête de journal dans le langage de requête LogQL. Vous pouvez afficher ce champ en cliquant sur Show Query.

![Dropdown-menu ](dropdown-menu.png)

