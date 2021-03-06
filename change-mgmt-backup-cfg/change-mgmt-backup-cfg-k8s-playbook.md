# Steps to change DatabaseBackup configuration in the Management Subsystem for Kubernetes Installations

The following are steps and a script to allow you to change the `DatabaseBackup` configuration in the Management Subsystem.

This is a workaround to a known issue and will be addressed in a future release.

**Note: This procedure is only compatible with Linux and Mac environments.**

## Scenarios when this procedure and script are needed

1. A customer wants to change their backup configuration from the default, post-install.
2. A customer wants to update their backup configuration to a different location (eg. different S3 bucket)
3. A customer has incorrectly configured their backup configuration and wants to remedy that (eg. incorrect S3 bucket name)

## FAQs
- Is the script rerunnable if something goes wrong: **Yes**
  
- Where can I find extra data for further analysis:
  - API Connect Operator logs
    - eg. `kubectl logs ibm-apiconnect-6c48b47cf8-w4ppl`
  - Pgbackrest stanza-create job logs
    - eg. `kubectl logs management-8359bacc-postgres-stanza-create-nh2fx`
  - Postgres Operator logs
    - eg. `kubectl logs postgres-operator-6947bd7769-hvxzm -c operator`
    - eg. `kubectl logs postgres-operator-6947bd7769-hvxzm -c apiserver`
  - Postgres Database logs
    - eg. `kubectl logs management-8359bacc-postgres-584c88dcd6-pn676 -c database`
  
- How long will this procedure be needed?: **This playbook is no longer needed in v10.0.1.0**

## Procedure

1. Ensure that you have `kubectl` installed from the location where you will run the script
   - https://kubernetes.io/docs/tasks/tools/install-kubectl/
   - Follow your Kubernetes provider instructions on how to find and connect to your Kubernetes cluster with `kubectl`

2. Verify you are able to run kubectl commands against your Kubernetes cluster
  ```
    $ kubectl get nodes
    NAME                            STATUS   ROLES    AGE   VERSION
    devtest-master.fyre.ibm.com     Ready    master   30h   v1.17.6
    devtest-worker-1.fyre.ibm.com   Ready    <none>   30h   v1.17.6
    devtest-worker-2.fyre.ibm.com   Ready    <none>   30h   v1.17.6
  ```
3. Copy the script below to the location where you wish to run the script from.
4. Name the script `change-mgmtbackup-cfg.sh`
5. In a terminal, run `chmod +x change-mgmtbackup-cfg.sh`
6. In a terminal, run `./change-mgmtbackup-cfg.sh <namespace> <mgmt_name>`
  - where `<namespace>` is the namespace where your Management Subsystem is deployed eg. `default`

## Summary of what the script does:

1. Checks current configuration
2. Scales down the API Connect Operator as we don't want the Operator reconciling any changes we make just yet
3. Creates the pgo-client deploy - this is a pod that allows us to interface with the Postgres Operator
4. Please refer to the Knowledge Center for procedure on configuring backup settings for Management subsystem: https://www.ibm.com/support/knowledgecenter/en/SSMNED_v10/com.ibm.apic.install.doc/tapic_db_backup_restore_apic.html
5. Once updated and customer agrees to proceed, the current postgres deployment is removed (no data is lost)
6. Scales up the API Connect Operator
7. The Operator reconciles everything back to normal, including creating the postgres deployment, with the new backup configuration

```
#!/bin/bash

ns=$1
[[ -z "$ns" ]] && echo "./change-mgmtbackup-cfg.sh <namespace> <mgmt_name>" && exit 1

mgmt_name=$2

print_current_cfg() {
    ns=$1
    mgmt_name=$2
    #Get all individual database backup values that are currently set in the mgmt CR
    s3provider=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.s3provider}")
    host=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.host}")
    path=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.path}")
    retries=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.retries}")
    credentials=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.credentials}")
    schedule=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.schedule}")

    [[ ! -z "$s3provider" ]]  && echo "S3 provider:  $s3provider"
    [[ ! -z "$host" ]]        && echo "Host:         $host"
    [[ ! -z "$path" ]]        && echo "Path:         $path"
    [[ ! -z "$retries" ]]     && echo "Retries:      $retries"
    [[ ! -z "$credentials" ]] && echo "Credentials:  $credentials"
    [[ ! -z "$schedule" ]]    && echo "Schedule:     $schedule"
}

abort() {
    echo "Answer was: $1"
    echo "Abort"
    exit 1
}

echo
echo "This script will allow you to change the DatabaseBackup configuration on your Management Subsystem"
echo

#If user gives the management cluster name check if it exists, else get the name of the management cluster name that is deployed in the namespace
if [ ! -z "$mgmt_name" ]; then
  kubectl get mgmt $mgmt_name -n $ns
  if [ "$?" -eq 1 ]; then
    echo
    echo "Management Subsystem \"$mgmt_name\" does not exist in namespace \"$ns\""
    echo
    abort
  fi
else
  mgmt_name=$(kubectl get mgmt -n $ns -o yaml | grep name: | head -n1 | awk -F ": " '{print $2}')
fi

echo "Management name: $mgmt_name"
echo


echo "Current DatabaseBackup configuration (if set):"
echo
print_current_cfg $ns $mgmt_name
echo

#Halt script from running and ask user do they want to continue
read -p "Changing the Management DatabaseBackup configuration will result in a short downtime of the database. There will be no loss of data. Proceed? (y/n) " -n 1 -r
echo

#If the user replies with anything other than Y or y stop the script
if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    abort $REPLY
fi

cluster_name=$(kubectl get mgmt $mgmt_name -n $ns  -o yaml | grep db: | awk -F ": " '{print $2}')
image_registry=$(kubectl get mgmt $mgmt_name -n $ns  -o yaml | grep imageRegistry: | awk -F ": " '{print $2}')
appVersion=$(kubectl get mgmt $mgmt_name -n $ns  -o yaml | grep appVersion: | awk -F ": " '{print $2}')
version=$(kubectl get mgmt $mgmt_name -n $ns  -o yaml | grep version: | awk -F ": " '{print $2}')
image_pull_secret=$(kubectl get mgmt $mgmt_name -n $ns  -o yaml | grep -A1 imagePullSecrets | tail -n1 | awk -F "- " '{print $2}')

if [ ! -z $appVersion ]; then
    version=$appVersion
fi

echo
echo "Management name:      $mgmt_name"
echo "Database name:        $cluster_name"
echo "Product version:      $version"
echo "Image Registry:       $image_registry"
echo "Image Pull Secret:    $image_pull_secret"
echo

if [[ "$version" == "10.0.1.0"* ]]; then
    pgo_client_image_tag="sha256:3295df5e00f11c072895627fdc5e84ca911c378b5ceb8b7c12ca55dfb7066891"
elif [[ "$version" == "10.0.0.0"* ]]; then
    pgo_client_image_tag="sha256:c728dee3458e38efced0474f95cd84f168f065a47761e483cd3551cdde8c824b"
else
    echo "Unsupported product version ${version} for ManagementCluster. Please contact IBM support." && exit 1
fi 

#Halt script from running and ask user do they want to continue
read -p "Please review the Management Subsystem CR and confirm these are correct. Proceed? (y/n) " -n 1 -r
echo

if [[ $REPLY =~ ^[Nn]$ ]]; then
    abort $REPLY
fi

echo "Creating copy of Management CR --> mgmt_cr.yaml"
kubectl get mgmt $mgmt_name -n $ns -o yaml > mgmt_cr.yaml

#Creating a pgo client pod so that it will allow us to interface with the postgres operator to allow us to create a new management cluster with the new desired/updated config
echo "Creating PGO client pod..."
cat <<EOF | kubectl create -f -
{
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "metadata": {
        "name": "pgo-client",
        "namespace": "$ns",
        "labels": {
            "vendor": "crunchydata"
        }
    },
    "spec": {
        "replicas": 1,
        "selector": {
                "matchLabels": {
                        "name": "pgo-client",
                        "vendor": "crunchydata"
                }
        },
        "template": {
            "metadata": {
                "labels": {
                    "name": "pgo-client",
                    "vendor": "crunchydata"
                }
            },
            "spec": {
                "imagePullSecrets": [
                    { "name": "$image_pull_secret" }
                ],
                "containers": [
                    {
                        "name": "pgo",
                        "image": "$image_registry/ibm-apiconnect-management-pgo-client@$pgo_client_image_tag",
                        "imagePullPolicy": "IfNotPresent",
                        "env": [
                            {
                                "name": "PGO_APISERVER_URL",
                                "value": "https://postgres-operator.$ns.svc:8443"
                            },
                            {
                                "name": "PGOUSERNAME",
                                "valueFrom": {
                                    "secretKeyRef": {
                                        "name": "pgouser-admin",
                                        "key": "username"
                                    }
                                }
                            },
                            {
                                "name": "PGOUSERPASS",
                                "valueFrom": {
                                    "secretKeyRef": {
                                        "name": "pgouser-admin",
                                        "key": "password"
                                    }
                                }
                            },
                            {
                                "name": "PGO_CA_CERT",
                                "value": "pgo-tls/tls.crt"
                            },
                            {
                                "name": "PGO_CLIENT_CERT",
                                "value": "pgo-tls/tls.crt"
                            },
                            {
                                "name": "PGO_CLIENT_KEY",
                                "value": "pgo-tls/tls.key"
                            }
                        ],
                        "volumeMounts": [
                            {
                                "name": "pgo-tls-volume",
                                "mountPath": "pgo-tls"
                            }
                        ]
                    }
                ],
                "volumes": [
                    {
                        "name": "pgo-tls-volume",
                        "secret": {
                            "secretName": "$mgmt_name-client",
                            "items": [
                                {
                                    "key": "tls.crt",
                                    "path": "tls.crt"
                                },
                                {
                                    "key": "tls.key",
                                    "path": "tls.key"
                                }
                            ]
                        }
                    }
                ]
            }
        }
    }
}
EOF
if [ "$?" -eq 1 ]; then
  echo "Error creating PGO client deployment"
  kubectl get deploy pgo-client -n $ns
  if [ "$?" -eq 0 ]; then
    echo
    read -p "Please confirm the pgo-client pod already exists and is running. Proceed? (y/n) " -n 1 -r
    echo

    if [[ ! $REPLY =~ ^[Yy]$ ]]
    then
        abort $REPLY
    fi
  else
    exit 1
  fi
fi

kubectl get pods -n $ns | grep pgo-client | grep -q Running
while [ "$?" -eq 1 ]; do
  echo "Waiting for pgo-client to start..."
  sleep 5
  kubectl get pods -n $ns | grep pgo-client | grep -q Running
done

pgo_client_pod_name=$(kubectl get pods -n $ns | grep pgo-client | tail -n1 | awk -F " " '{print $1}')

echo
echo "PGO Client pod name: $pgo_client_pod_name"
echo

echo "Please update the Management Subystem backup configuration now"
echo
echo "You can change your Management CR settings via 'kubectl edit mgmt <mgmt_name>'"
echo "Alternatively, you can update your management_cr.yaml file and update with 'kubectl apply management_cr.yaml'"
echo
read -p "Proceed when the configuration is updated. Proceed? (y/n) " -n 1 -r
echo

if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    abort $REPLY
fi

REPLY=""
while [[ ! $REPLY =~ ^[Yy]$ ]]; do
    if [[ $REPLY =~ ^[Cc]$ ]]; then
        abort $REPLY
    fi

    echo
    echo "New DatabaseBackup configuration:"
    echo
    print_current_cfg $ns $mgmt_name
    echo

    credentials=$(kubectl get mgmt $mgmt_name -n $ns -o jsonpath="{..databaseBackup.credentials}")
    if [ ! -z "$credentials" ]; then
        kubectl get secret $credentials -n $ns > /dev/null 2>&1 

        if [  "$?" -eq 1 ]; then
            echo "WARNING: could not find secret \"$credentials\" in namespace \"$ns\""
            echo
            echo "Please create credentials secret now"
            echo
        fi
    fi

    read -p "Please review the backup configuration and confirm these are correct. Reconfiguration will begin after proceeding. Proceed? (y/n/c n=recheck c=abort) " -n 1 -r
    echo
done

read -p "Restart database? This will result in a brief downtime of the database. Proceed with restarting database? (y/n) " -n 1 -r
echo

if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    abort $REPLY
fi

#Scaling down the apic operator to 0. This is because if we have the operator running it will not allow us to delete a management cluster deployment as it will constantly redeploy the old deployment we want to change
echo "Scaling down APIC Operator..."
kubectl scale deploy ibm-apiconnect --replicas=0 -n $ns
if [ "$?" -eq 1 ]; then
  echo "Error scaling down ibm-apiconnect deployment"
  kubectl get deploy ibm-apiconnect
  if [ "$?" -eq 1 ]; then
    echo
    read -p "If running the APIC Operator locally, please manually stop the APIC Operator now. Proceed when stopped. Proceed? (y/n) " -n 1 -r
    echo

    if [[ ! $REPLY =~ ^[Yy]$ ]]
    then
        echo "Answer was: $REPLY"
        echo
        abort
    fi
  else
    exit 1
  fi
fi

#Stopping database pods because currently there are no way of updating them so we need to delete and recreate
echo
echo "Stopping database pods..."
kubectl exec -it $pgo_client_pod_name -n $ns -- pgo delete cluster $cluster_name -n $ns --keep-data --no-prompt
if [ "$?" -eq 1 ]; then
  echo "Error stopping database pods" && exit 1
  kubectl scale deploy ibm-apiconnect --replicas=1 -n $ns
  [[ "$?" -eq 1 ]] && echo "Error scaling up ibm-apiconnect deployment" && exit 1
fi

kubectl get pgcluster $cluster_name -n $ns > /dev/null 2>&1 
while [ "$?" -eq 0 ]; do
  echo "Waiting for database to be stopped..."
  sleep 5
  kubectl get pgcluster $cluster_name -n $ns > /dev/null 2>&1 
done

echo "Database stopped"
echo
echo "Scaling up APIC Operator..."
echo

#Scaling the apiconnect operator back up. When the operator has scaled back up it will see that it is missing postgres deployments and will recreate them
kubectl scale deploy ibm-apiconnect --replicas=1 -n $ns
if [ "$?" -eq 1 ]; then
  echo "Error scaling up ibm-apiconnect deployment"
  kubectl get deploy ibm-apiconnect -n $ns
  if [ "$?" -eq 1 ]; then
    read -p "If running the APIC Operator locally, please manually start the APIC Operator. Proceed when started. Proceed? (y/n) " -n 1 -r
    echo

    if [[ ! $REPLY =~ ^[Yy]$ ]]
    then
        abort $REPLY
    fi
  else
    exit 1
  fi
fi

kubectl get pgcluster $cluster_name -n $ns > /dev/null 2>&1 
while [ "$?" -eq 1 ]; do
  echo "Waiting for database to be started..."
  sleep 5
  kubectl get pgcluster $cluster_name -n $ns > /dev/null 2>&1 
done
echo "Database started"
echo
echo "Success"
exit 0
```