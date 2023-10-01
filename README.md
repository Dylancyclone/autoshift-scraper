

# Overview
Script aimed at scraping SHiFT Codes from websites, initially focused at those hosted on https://mentalmars.com, such as: 
 - https://mentalmars.com/game-news/borderlands-3-golden-keys/

Instead of publishing this as part of [Fabbi's autoshift](https://github.com/Fabbi/autoshift), this is aimed at publishing a machine readable file that can be hit by autoshift.  This reduces the load on mentalmars as it's likely not ok to have swarms of autoshifts scraping their website.

This has been setup with the intent that other webpages could be scraped. The  Python Dictionary `webpages` can be used to customise the webpage, the tables and their contents. This may need adjusting as mentalmars' website updates over time also

TODO List: 
- [x] Scrape mentalmars
- [x] output into a autoshift compatible json file format
- [ ] change to find `table` tags in `figure` tags to reduce noise in webpage
- [x] publish to GitHub [here](https://raw.githubusercontent.com/ugoogalizer/autoshift-codes/main/shiftcodes.json)
- [x] dockerise and schedule
- [ ] identify expired codes on website (strikethrough)
- [ ] identify expired codes by date


# Use
## Command Line Use
``` bash
# If only generating locally
python ./autoshift-scraper.py 

# If pushing to GitHub:
python ./autoshift-scraper.py --user GITHUB_USERNAME --repo GITHUB_REPOSITORY_NAME --token GITHUB_AUTHTOKEN

# If scheduling: 
python ./autoshift-scraper.py --schedule 5 # redeem every 5 hours
```

## Docker Use

The following docker environment variables are in use: 

| Environment Variable | Use |
| -------------------- | --- |
| GITHUB_USER | The username that owns the GitHub repo to commit to | 
| GITHUB_REPO | The name of the GitHub repository to commit to
| GITHUB_TOKEN | The GitHub fine-grained personal access token -- see below for more details | 
| PARSER_ARGS | (Optional) Additional parameters to pass in, like "--schedule 2 --verbose" |

Example: 
``` bash
docker run -d -t -i \
-e GITHUB_USER='ugoogalizer' \ 
-e GITHUB_REPO='autoshift-codes' \
-e GITHUB_TOKEN='github_pat_***' \
-e PARSER_ARGS='--verbose --schedule 2'+
-v autoshift:/autoshift/data \
--name autoshift-scraper \
ugoogalizer/autoshift-scraper:latest
```
Example localhost build image: 
``` bash
docker run -d -t -i \
-e GITHUB_USER='ugoogalizer' \
-e GITHUB_REPO='autoshift-codes' \
-e GITHUB_TOKEN='github_pat_***' \
-e PARSER_ARGS='--verbose --schedule 2' \
-v autoshift:/autoshift/data \
--name autoshift-scraper \
localhost/autoshift-scraper:latest

```

## Kubernetes Use

Example Deployment file

``` yaml

--- # deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: autoshift-scraper
  name: autoshift-scraper
#  namespace: autoshift
spec:
  selector:
    matchLabels:
      app: autoshift-scraper
  revisionHistoryLimit: 0
  template:
    metadata:
      labels:
        app: autoshift-scraper
    spec:
      containers:
        - name: autoshift-scraper
          # Fix version so it doesn't auto-update
          image: ugoogalizer/autoshift-scraper:0.6
          imagePullPolicy: IfNotPresent
          env:
            - name: GITHUB_USER
              value: "ugoogalizer"
            - name: GITHUB_REPO
              value: "autoshift-codes"
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: autoshift-scraper-secret
                  key: githubtoken
            - name: PARSER_ARGS
              value: "--schedule 2"
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: "100m"
              memory: "500Mi"
          volumeMounts:
            - mountPath: /autoshift-scraper/data
              name: autoshift-scraper-pv
      volumes:
        - name: autoshift-scraper-pv
          # If this is NFS backed, you may have to add the nolock mount option to the storage class
          persistentVolumeClaim:
            claimName: autoshift-scraper-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
# If this is NFS backed, you may have to add the nolock mount option to the storage class
metadata:
  name: autoshift-scraper-pvc
#  namespace: autoshift
spec:
  storageClassName: managed-nfs-storage-retain
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi


# kubectl create namespace autoshift
# kubectl config set-context --current --namespace=autoshift
# kubectl create secret generic autoshift-scraper-secret --from-literal=githubtoken='XXX' 

# To get the username and password use: 
# kubectl get secret autoshift-scraper-secret -o jsonpath="{.data.githubtoken}" | base64 -d

```

# Configuring GitHub connectivity

Need to create a new fine-grained personal access token, with access to the only the destination repo and Read & Write access to "Contents"

The token should look something like 

```
github_pat_11p9ou8easrhsgp98sepfg97gUS98hu7ASFuASFDNOANSFDASF ... (but much longer)
```


# Setting up development environment


## Original setup

``` bash
# setup venv
python3 -m venv .venv
source ./.venv/bin/activate

# install packages
pip install requests bs4 html5lib PyGithub APScheduler

pip freeze > requirements.txt
```

## Docker Container Image Build

``` bash

# Once off setup: 
git clone TODO

# Personal parameters
export HARBORURL=harbor.test.com

git pull

#Set Build Parameters
export VERSIONTAG=0.6

#Build the Image
docker build -t autoshift-scraper:latest -t autoshift-scraper:${VERSIONTAG} . 

#Get the image name, it will be something like 41d81c9c2d99: 
export IMAGE=$(docker images -q autoshift-scraper:latest)
echo ${IMAGE}

#Tag and Push the image in harbor
docker login ${HARBORURL}:443
docker tag ${IMAGE} ${HARBORURL}:443/autoshift/autoshift-scraper:latest
docker tag ${IMAGE} ${HARBORURL}:443/autoshift/autoshift-scraper:${VERSIONTAG}
docker push ${HARBORURL}:443/autoshift/autoshift-scraper:latest
docker push ${HARBORURL}:443/autoshift/autoshift-scraper:${VERSIONTAG}

```