# Installing 3scale AMP 2.1 on OCP 3.6 on ec2


You will need:

- a running instance with 8GB RAM minimum (recommended 16GB) and RHEL

- **`<PUBLIC_DNS>`**: (e.g. `ec2-54-123-456-78.compute-1.amazonaws.com`)

- **`<PUBLIC_IP>`**: (e.g. `54.123.456.78`)

## Set up OpenShift cluster 

### References

- https://github.com/openshift/origin/blob/master/docs/cluster_up_down.md#linux

### Install and run Docker

```
sudo yum-config-manager --enable rhui-REGION-rhel-server-extras
sudo yum install docker docker-registry -y
```

`/etc/containers/registries.conf`:
```
registries = ['172.30.0.0/16']
```
```
sudo systemctl start docker
sudo systemctl status docker
```

### Install OC tools

**OCP:** https://mirror.openshift.com/pub/openshift-v3/clients/3.6.173.0.21/linux/oc.tar.gz

Example:
```
sudo yum install wget -y
wget https://mirror.openshift.com/pub/openshift-v3/clients/3.6.173.0.21/linux/oc.tar.gz
tar xzvf oc.tar.gz
mv oc /usr/bin/
rm -rf oc.tar.gz
```

### Start the cluster

```
oc cluster up --public-hostname=<PUBLIC_DNS> --routing-suffix=<PUBLIC_IP>.xip.io --host-data-dir=/home/ec2-user/myoccluster1
```

Check out the console:
`https://<PUBLIC_DNS>:8443`

## Deploy 3scale AMP

### Create persistent volumes

```
sudo su

mkdir -p  /var/lib/docker/pv/{01..04}
chmod g+w /var/lib/docker/pv/{01..04}
chcon -Rt svirt_sandbox_file_t /var/lib/docker/pv/
```

(`pv.yml` - copy from resources folder)

```
oc login -u system:admin

oc new-app --param PV=01 -f pv.yml
oc new-app --param PV=02 -f pv.yml
oc new-app --param PV=03 -f pv.yml
oc new-app --param PV=04 -f pv.yml

oc get pv
```

(`amp.yml` - copy from resources folder)


### Login as developer and start AMP with template


```
oc login https://<PUBLIC_DNS>:8443 --insecure-skip-tls-verify

oc new-project 3scale-amp

oc new-app --file amp.yml --param TENANT_NAME=3scale --param WILDCARD_DOMAIN=<PUBLIC_IP>.xip.io >> /tmp/3scale_amp_provision_details.txt
```

```

cat /tmp/3scale_login_details.txt


--> Deploying template "3scale-amp/system" for "amp.yml" to project 3scale-amp

     system
     ---------
     Login on Login on https://vb-admin.54.86.18.216.xip.io as admin/gu8edykg    <===== LOGIN with these credentials

     * With parameters:
        * AMP_RELEASE=er3
        * ADMIN_PASSWORD=gu8edykg # generated
        * ADMIN_USERNAME=admin
        * APICAST_ACCESS_TOKEN=rthdeuql # generated
        * ADMIN_ACCESS_TOKEN=4o2txf0v4e3wgvtw # generated
        * WILDCARD_DOMAIN=<PUBLIC_IP>.xip.io
        * SUBDOMAIN=3scale
        * MySQL User=mysql
        * MySQL Password=qfnt75jf # generated
        * MySQL Database Name=system
        * MySQL Root password.=7dhquse7 # generated
        * SYSTEM_BACKEND_USERNAME=3scale_api_user
        * SYSTEM_BACKEND_PASSWORD=a3i3n7by # generated
        * REDIS_IMAGE=rhscl/redis-32-rhel7:3.2-5.3
        * SYSTEM_BACKEND_SHARED_SECRET=s4wpndxj # generated
```

### RESUME PODS

```
1. Resume database tier pods

for x in backend-redis system-memcache system-mysql system-redis zync-database; do echo Resuming dc:  $x; sleep 2; oc rollout resume dc/$x; done

Verify all pods are running

oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
backend-redis-1-q2hnc     1/1       Running   0          53s
system-memcache-1-ggjgm   1/1       Running   0          49s
system-mysql-1-pg7rm      1/1       Running   0          1m
system-redis-1-klthg      1/1       Running   0          44s
zync-database-1-w66qf     1/1       Running   0          52s
```

```
2. Resume backend listener and worker deployments:

for x in backend-listener backend-worker; do echo Resuming dc:  $x; sleep 2; oc rollout resume dc/$x; done
```

```
3. Resume the system-app and its two containers:

oc rollout resume dc/system-app
```

```
4. Resume additional system and backend application utilities:

for x in system-resque system-sidekiq backend-cron system-sphinx; do echo Resuming dc:  $x; sleep 2; oc rollout resume dc/$x; done
```

```
5. Resume apicast gateway deployments:

for x in apicast-staging apicast-production; do echo Resuming dc:  $x; sleep 2; oc rollout resume dc/$x; done
```

```
6. Resume remaining deployments:

for x in apicast-wildcard-router zync; do echo Resuming dc:  $x; sleep 2; oc rollout resume dc/$x; done
```