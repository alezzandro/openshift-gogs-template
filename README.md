# openshift-gogs-template
Deploy your own git server on OpenShift v3 with Gogs container!

With this document I'll show you the easy steps to deploy an integrated git server on your existing OpenShift v3 platform.

I'll describe you how to deploy a Gogs container with/without internal mysql server (you'll be able to easily replace mysql with your own favourite sql server).

First of all we need to download the latest git version of this repo:
```
[alex@freddy ~]$ git clone https://github.com/alezzandro/openshift-gogs-template
```

The we can start to login to our OpenShift platform:
```
[alex@freddy ~]$ oc login
Authentication required for https://192.168.121.1:8443 (openshift)
Username: admin
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
  * glusterfs
  * management-infra
  * openshift
  * openshift-infra
  * sample-project

Using project "default".
```
Then we can create a new project for holding all the resources we'll create:
```
[alex@freddy ~]$ oc new-project gogs
Now using project "gogs" on server "https://192.168.121.1:8443".

You can add applications to this project with the 'new-app' command. For example, try:

    $ oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-hello-world.git

to build a new hello-world application in Ruby.
```

We can now start creating all the needed resources, first of all we'll create a ServiceAccount for handling Gogs' container particular needs: 
```
[alex@freddy gogs-template]$ oc create -f gogs-sa.yml 
serviceaccount "gogs" created
```

And then we give it all the needed rights:
```
[alex@freddy gogs-template]$ oadm policy add-scc-to-user privileged system:serviceaccount:gogs:gogs
```


At this point you have two option:

1. Use Gogs standalone with sqlite support <br />
2. Use Gogs with mysql server support

## 1. Use Gogs standalone with sqlite support

We can now import the template we want to use:
```
[alex@freddy gogs-template]$ oc create -f gog-standalone-template.yml 
template "gogs-mysql-template" created
```
And then  use it to create a new-app, please note how we actually override APPLICAITON_DOMAIN variable:
```
[alex@freddy gogs-template]$ oc new-app --template=gogs-standalone-template -p APPLICATION_DOMAIN=gogs-standalone.192.168.121.1.xip.io
--> Deploying template gogs-standalone-template for "gogs-standalone-template"
     With parameters:
      Application Hostname=gogs-standalone.192.168.121.1.xip.io
--> Creating resources ...
    route "gogs-standalone" created
    service "gogs-standalone" created
    deploymentconfig "gogs-standalone" created
--> Success
    Run 'oc status' to view your app.
```

As suggested by the command's output we can then monitor the running pods:
```
[alex@freddy gogs-template]$ oc get pods
NAME                       READY     STATUS              RESTARTS   AGE
gogs-standalone-1-deploy   1/1       Running             0          4s
gogs-standalone-1-pmmu4    0/1       ContainerCreating   0          1s
[alex@freddy gogs-template]$ oc get pods
NAME                      READY     STATUS    RESTARTS   AGE
gogs-standalone-1-pmmu4   1/1       Running   0          3s
```

And then ensure that the route is correctly created:
```
[alex@freddy gogs-template]$ oc get route
NAME              HOST/PORT                              PATH      SERVICE                TERMINATION   LABELS
gogs-standalone   gogs-standalone.192.168.121.1.xip.io             gogs-standalone:3000                 app=gogs-standalone
```

We can now navigate to http://gogs-standalone.192.168.121.1.xip.io and proceed with Gogs installation using sqlite3 support, please remember to set Application URL: to your route address.

<b>
Please note: the current Gogs template is not shipped with any persistent volume configuration, so YOU'll need to configure a persistent storage for the configured DeploymentConfig once you'll deploy the app from template.
</b>

## 2. Use Gogs with mysql server support

We can now import the template we want to use:
```
[alex@freddy gogs-template]$ oc create -f gogs-mysql-template.yml 
template "gogs-mysql-template" created
```
And then  use it to create a new-app, please note how we actually override APPLICAITON_DOMAIN variable:
```
[alex@freddy gogs-template]$ oc new-app --template=gogs-mysql-template -p APPLICATION_DOMAIN=gogs-mysql.192.168.121.1.xip.io
--> Deploying template gogs-mysql-template for "gogs-mysql-template"
     With parameters:
      Application Hostname=gogs-mysql.192.168.121.1.xip.io
      Database Name=default
      Database User=gogs
      Database Password=4TE8Tve47Yd0gW23 # generated
      Memory Limit (MySQL)=512Mi
      Database Service Name=mysql
--> Creating resources ...
    route "gogs" created
    service "gogs" created
    service "mysql" created
    deploymentconfig "gogs" created
    deploymentconfig "mysql" created
--> Success
    Run 'oc status' to view your app.
```
As suggested by the command's output we can then monitor the running pods:
```
[alex@freddy gogs-template]$ oc get pods
NAME             READY     STATUS    RESTARTS   AGE
gogs-1-uha0z     1/1       Running   0          5s
mysql-1-deploy   1/1       Running   0          8s
mysql-1-mq78t    0/1       Running   0          5s

[alex@freddy gogs-template]$ oc get pods
NAME            READY     STATUS    RESTARTS   AGE
gogs-1-uha0z    1/1       Running   0          21s
mysql-1-mq78t   1/1       Running   0          21s
```

After that we can take a look to the password it has automatically generated:
```
[alex@freddy gogs-template]$ oc describe pod mysql-1-mq78t|grep PASS
      MYSQL_PASSWORD:		4TE8Tve47Yd0gW23
      MYSQL_ROOT_PASSWORD:	4TE8Tve47Yd0gW23
```
And then ensure that the route is correctly created:
```
[alex@freddy gogs-template]$ oc get route
NAME      HOST/PORT                         PATH      SERVICE     TERMINATION   LABELS
gogs      gogs-mysql.192.168.121.1.xip.io             gogs:3000                 app=gogs
```

We can now navigate to http://gogs-mysql.192.168.121.1.xip.io and configure for db: mysql:3306, username: gogs, password: (the previous one), database: default. Application URL: your route.

<b>
Please note: the current Gogs template is not shipped with any persistent volume configuration, so YOU'll need to configure a persistent storage for the configured DeploymentConfig (for both mysql and gogs) once you'll deploy the app from template.
</b>

That's all!

If you want to improve this how-to and/or the template please feel free to make a pull request!
