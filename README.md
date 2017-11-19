# Openshift Logging EFK using GlusterFS Persistent Volume
- I am using this case Openshift Enterprise 3.5 and GlusterFS 3.2 Container Native Storage.
- My glusterfs services running on pods.In this enable Cluster Metric existing Openshift Cluster.


##### GlusterFS Preperation For Cluster Metric
-  Before you have deploy cluster logging on existing Openshift Cluster you are prepare glusterfs Persistent backend.Because Openshift Enterprise 3.5 ansible playbook only install Metric Cluster on existing Persistent volume.If you deploy first Logging cluster there will a a few issue creating elasticsearch persistent volume.Whereas glusterfs volume security at first elasticsearch pod won't running state.
###### Start with deploy Cluster Logging using ansible playbook.

If you have not openshift-ansible playbook. you can get it from Github from version 3.5

`# git clone https://github.com/openshift/openshift-ansible.git`

##### You can using sample logging parameter in your inventory file as below
###########LOGGING DEFINITION###############

openshift_logging_install_logging=true
openshift_logging_use_ops=true
openshift_logging_namespace=logging
openshift_logging_master_public_url=openshift.lp.int                      
openshift_logging_curator_nodeselector={"region":"infra"}
openshift_logging_kibana_hostname=kibana.app.lp.int
openshift_logging_kibana_nodeselector={"region":"infra"}
openshift_logging_kibana_ops_hostname=kibana-ops.app.lp.int
openshift_logging_fluentd_nodeselector={"region":"infra"}
openshift_logging_es_pv_selector={"logging":"elastic"}
openshift_logging_es_pvc_size=10G
openshift_logging_es_pvc_prefix=logging-es
openshift_logging_es_storage_group=65000
openshift_logging_es_nodeselector={"region":"infra"}
openshift_logging_es_ops_allow_cluster_reader=true
openshift_logging_es_ops_pv_selector={"logging-ops":"elastic-ops"}
openshift_logging_es_ops_pvc_size=10G
openshift_logging_es_ops_pvc_prefix=logging-es-ops
openshift_logging_es_ops_storage_group=65001
openshift_logging_es_ops_nodeselector={"region":"infra"}
openshift_logging_kibana_ops_nodeselector={"region":"infra"}
openshift_logging_curator_ops_nodeselector={"region":"infra"}

- Run only logging playbooks

`ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/openshift-logging.yml -e deployment_type=openshift-enterprise`


You can also look at logging aggregating from Openshift Documentation.
`<link>`: https://docs.openshift.com/container-platform/3.5/install_config/aggregate_logging.html

After deployment complete elasticsearch pod will not running state.
At this point we have to prepare glusterfs persistent backend.
-  Create gluster end point for elasticsearch

`oc create -f logging-es-endpoint.yaml`

- You do not need create exstra service because ansible playbook create all services

After It's done we have to get fsgroup from elasticsearch pod as below.Because elasticsearch application has to required permssion to write to the glusterfs persistent Volume

`# export GID=$(oc get po --selector="logging-es-c8quc3va-1-5psdg=openshift-infra" -o go-template --template='{{printf "%.0f" ((index .items 0).spec.securityContext.fsGroup)}}')`

###### NOTE: if you can not get fsGroup as above command you can manualy find it `oc get -o pod logging-es-c8quc3va-1-5psdg -o yaml | grep fsGroup`

- Create Glusterfs volume for elasticsearch pod

`# heketi-cli volume create --size=10 --name=logging-es --gid=${GID}`

- After create gluster volume persistent volume as below.

###### NOTE: You have to replace labels, endpoints and path

`# oc create -f pvlogging-es.json`

- Same way gluster end point for elasticsearch-ops

`logging-es-ops-endpoint.yaml`

You can use same fsGroup ID for newly create glusterfs volume

- Create Glusterfs volume for elasticsearch-ops pod

`heketi-cli volume create --size=10 --name=logging-es-ops --gid=${GID}`

- Create Persistent volume for elasticsearch-ops

`oc create -f pv-ops-logging.yaml`
