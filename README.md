# k8s-nginx-clientes-mongodb

Para rodar a aplicação:

> kubectl create -f https://tonanuvem.github.io/k8s-clientes-nginx-mongodb/cliente-svc-deploy.yaml

Os seguintes serviços serão criados:

```
> ~/k8s-nginx-clientes-mongodb$ kubectl get svc
```

|NAME          |TYPE       |CLUSTER-IP     |EXTERNAL-IP  |PORT(S)         |AGE|
| --- | --- | --- | ---| --- | ---|
|backend        |ClusterIP    |10.106.72.158   |<none>        |5000/TCP   |2m40s
|frontend       |NodePort    |10.99.142.117   |<none>        |80:32000/TCP     |2m40s
|mongo          |ClusterIP   |10.102.58.27    |<none>        |27017/TCP        |2m40s
|mongoexpress   |NodePort    |10.100.3.61     |<none>        |8081:32081/TCP   |2m40s
 
Código fonte da aplicação: https://github.com/tonanuvem/nginx-clientes-microservice-mongodb

<hr>

* Portworks

Versao 1.19.2 ( verificar com kubectl version --short | awk -Fv '/Server Version: / {print $3}' )

> lsblk
> VER=`kubectl version --short | awk -Fv '/Server Version: /{print $3}'`
> curl -L -s -o px-spec.yaml "https://install.portworx.com/2.6?mc=false&kbver=${VER}&b=true&s=%2Fdev%2Fxvdb&c=px-fiap&stork=true&st=k8s"
> kubectl apply -f px-spec.yaml

c=px-fiap specifies the cluster name
b=true specifies to use internal etcd
kbVer=${VER} specifies the Kubernetes version
s=/dev/xvdb specifies the block device to use
 
ou

> kubectl apply -f 'https://install.portworx.com/2.6?mc=false&kbver=1.19.2&oem=esse&user=075ebe88-f8e2-11ea-a2c5-c24e499c7467&b=true&c=px-cluster-3ae228df-0ebe-4a69-bbf5-4c6bdc30cc18&stork=true&lh=true&st=k8s'

> kubectl get pods -o wide -n kube-system -l name=portworx
It can take a few minutes for Portworx to complete initialization
When all nodes are Ready 1/1
Demora uns 4 min.

> PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')

> kubectl exec -it $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status

|Nodes: 3 node(s) with storage (3 online)
|Global Storage Pool
|Total Used    	:  5.6 GiB
|Total Capacity	:  90 GiB

- Agora temos um cluster Portworx de 3 nós ativado!
- Vamos mergulhar em nosso status de cluster.
- Todos os 3 nós estão online e usam nomes de nós Kubernetes como os IDs de nós Portworx.
- Observe que Portworx agrupou o dispositivo de bloco de cada nó em um cluster de armazenamento de 3X.
- O Portworx detectou o tipo de mídia do dispositivo de bloco como SSD e criou um pool de armazenamento para isso.

> kubectl -n kube-system describe pods $PX_POD
 
Events:
   |Type     |Reason                             |Age                     |From                  |Message
   |----     |------                             |----                    |----                  |-------
   |Normal   |Scheduled                          |7m57s                   |default-scheduler     |Successfully assigned kube-system/portworx-qxtw4 to k8s-node-2
   |...
   |Warning  |Unhealthy                          |5m15s (x15 over 7m35s)  |kubelet, k8s-node-2   |Readiness probe failed: HTTP probe failed with statuscode: 503
   |Normal   |NodeStartSuccess                   |5m7s                    |portworx, k8s-node-2  |PX is ready on this node

<hr>

* Persistent Volume Claim (PVC) e Storage Class (SC)

Exemplo de uso: https://docs.portworx.com/portworx-install-with-kubernetes/storage-operations/create-pvcs/dynamic-provisioning/

> kubectl create -f https://tonanuvem.github.io/k8s-clientes-nginx-mongodb/pvc-storageclass.yaml

> kubectl describe pvc mongo-pvc

Successfully provisioned volume pvc-0ebc5d99-0d7a-41d9-a218-6d02e9056e38 using kubernetes.io/portworx-volume

> kubectl get pv

Verificar o volume criado dinamicamente.

<hr>

* MongoDB

Criar o Deploy e SVC

> POD=`kubectl get pods | grep 'mongo-' | awk '{print $1}'`

> echo $POD
mongo-7d5bdbdbb-jktm7

> kubectl exec -it $POD -- mongo --host localhost

db.ships.insert({name:'USS Enterprise-D',operator:'Starfleet',type:'Explorer',class:'Galaxy',crew:750,codes:[10,11,12]})

db.ships.insert({name:'Narada',operator:'Romulan Star Empire',type:'Warbird',class:'Warbird',crew:65,codes:[251,251,220]})

db.ships.find().pretty()

db.ships.find({}, {name:true, _id:false})

db.ships.findOne({'name':'USS Enterprise-D'})

> NODE=`kubectl get pods -l app=mongodb -o wide | grep -v NAME | awk '{print $7}'`

> kubectl cordon ${NODE}

> POD=`kubectl get pods -l app=mongodb -o wide | grep -v NAME | awk '{print $1}'`

> kubectl delete pod ${POD}

> watch kubectl get pods -l app=mongo -o wide

Verify data is still available

 > POD=`kubectl get pods | grep 'px-mongo-' | awk '{print $1}'`
 
 > kubectl exec -it $POD -- mongo --host px-mongo-mongodb

db.ships.insert({name:'USS Enterprise-D',operator:'Starfleet',type:'Explorer',class:'Galaxy',crew:750,codes:[10,11,12]})

db.ships.insert({name:'Narada',operator:'Romulan Star Empire',type:'Warbird',class:'Warbird',crew:65,codes:[251,251,220]})

db.ships.find().pretty()

db.ships.find({}, {name:true, _id:false})

db.ships.findOne({'name':'USS Enterprise-D'})

- Inspect the volume

You can run pxctl commands to inspect your volume:

> VOL=`kubectl get pvc | grep px-mongo-pvc | awk '{print $3}'`

> PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')

> kubectl exec -it $PX_POD -n kube-system -- /opt/pwx/bin/pxctl volume inspect ${VOL}

> kubectl get pvc px-mongo-pvc

Expand the volume: Since we added the allowVolumeExpansion: true attribute to our storage class you can expand the PVC by editing the px-mongo-pvc.yaml file and then re-applying this file using kubectl.

> sed -i 's/10Gi/20Gi/g' px-mongo-pvc.yaml

> kubectl apply -f px-mongo-pvc.yaml

Inspect the volume and verify that it now has 20Gi capacity:

> kubectl get pvc px-mongo-pvc

As you can see the volume is now expanded and our MongoDB database didn't require restarting.

> kubectl get pods

<hr>

* Snapshot de um Volume:

> kubectl create -f https://tonanuvem.github.io/k8s-clientes-nginx-mongodb/vol_snapshot.yaml

You can see the snapshots using the following command:

> kubectl get volumesnapshot,volumesnapshotdatas

Now we're going to go ahead and do something stupid:

> POD=`kubectl get pods | grep 'px-mongo-' | awk '{print $1}'`

> kubectl exec -it $POD -- mongo --host px-mongo-mongodb

db.ships.remove({})

db.ships.find({}, {name:true, _id:false})

exit

- Restore the snapshot and see your data is still there

Snapshots are just like volumes so we can go ahead and use it to start a new instance of MongoDB. Note here that we're leaving the old instance to carry on with it's version of the volume and we're creating a brand new instance of MongoDB with the snapshot data!

>  

<hr>

* Helm
 
> helm install --name mongodb --set auth.enabled=false,service.portName=mongo,persistence.existingClaim=px-mongo-pvc bitnami/mongodb
 
 link: https://hub.helm.sh/charts/bitnami/mongodb
