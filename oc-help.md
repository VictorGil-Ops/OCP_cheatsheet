# Application components

	+Build config 
		oc get bc
	+ Deployment config
		oc get dc
	+ Image stream
		oc get is
	+ Deployment
		oc get deployment
	+ Pod
		oc get pod
	+ Replication controler
		oc get rc
	+ Service
		oc get service
	+ Image
		oc get image
	+ Route
		oc get route


# Common commands:

	oc status
	oc api-resources
	oc logs pod <mypod>
	oc get hpa
	oc describe pod <mypod> || oc describe dc <mydc>
	oc get services --sort-by=.metadata.name
	oc delete all -l app=tomcat
	oc delete pod <mypod> --grace-period=0
	oc export bc,dc,is,svc --as-template=myapp.yaml
	oc get -o yaml --export all > project.yaml
	oc get -o yaml --export dc,sv config-service > dc_config-service.yaml
	oc get dc -o template --template '{{range .items}}{{.metadata.name}},{{index .metadata.labels "app"}}{{"\n"}}{{end }}'
	oc get dc -o template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end }}'
	oc get dc <dc> -o=jsonpath='{range .items[*]}{.spec.template.spec.containers[].env[*].name}'
	oc edit dc micro-front
	oc rollout latest dc/<mydc> -n <namespace>
	oc rollout undo dc/<mydc> -n <namespace>
	oc rollback dc/configuration-service --to-version=22
	
	## replace
	oc create secret generic privkey --from-file=/opt/privkey --dry-run -o yaml | oc replace -f -
	
	## env values
	oc set env dc configuration-service --list

	
## DC
	
	+ autoscale
	oc get hpa
	oc patch dc gateway-g -p '{"metadata":{"labels":{"alfalfa":"true"}}}' -n project-pro
	oc patch dc gateway-g -p '{"spec":{"template":{"metadata":{"labels":{"alfalfa":"true"}}}}}' -n project-pro
	oc autoscale dc/gateway-g --min 5 --max 45 --cpu-percent=100 -n project-pro
	oc scale dc gateway-g --replicas=5 -n san-enquires-to-lake-pro
	
	+ Revisar log level y estado de los DC
	
	oc get dc -o=custom-columns=NOMBRE:'.metadata.name',LOG_ROOT:'.spec.template.spec.containers[].env[?(@.name=="ROOT_LOG")].value',TECH_LOG:'.spec.template.spec.containers[].env[?(@.name =="TECH_LOG")].value',DARWIN_LOGGING:'.spec.template.spec.containers[].env[?(@.name =="DARWIN_LOGGING_LOGLEVEL_ROOT")].value',DARWIN_REGION:'.spec.template.spec.containers[].env[?(@.name =="DARWIN_REGION")].value'
	
	+ Estado DC

	oc get dc -o=custom-columns=NOMBRE:'.metadata.name',ESTADO_DEL_DC:'status.conditions[1].message',ESTADO:'status.conditions[1].status' | grep -i false

	oc get dc -o=custom-columns=NOMBRE:'.metadata.name',ESTADO_DEL_DC:'status.conditions[1].message' | grep failed
	
## PODS
	
	oc get pods -w
	oc get pod --all-namespaces -o wide
	oc get pod --all-namespaces --no-headers=true | awk '{print $1","$2}' > namespaces.csv
	
	# List pods Sorted by Restart Count
	oc get pods --sort-by='.status.containerStatuses[0].restartCount'
	
	oc get pods --field-selector=status.phase=Running
	
	oc get pods --selector=app=cassandra -o jsonpath='{.items[*].metadata.labels.version}'
	
	oc get pods --show-labels
	
	# Scale
	oc scale dc <mydc> --replicas=0
	
	# Estado de los pod
	oc get pod -o=json | jq '.items[]|select(any( .status.containerStatuses[]; .state.waiting.reason=="ImagePullBackOff"))|.metadata.name'
	oc get pod -o=json | jq '.items[]|select(any( .status.containerStatuses[]; .state.terminated.reason=="Completed"))|.metadata.name'
	oc get pod -o=json | jq '.items[]|select(any( .status.containerStatuses[]; .state.waiting.reason=="Running"))|.metadata.name'
	
	# eliminar pods estado distinto a RUNNING
	for pod in $(oc get pods --no-headers --field-selector=status.phase!=Running| awk '{print $1}'); do oc delete pod --grace-period=1 ${pod}; done;

	# NAMESPACE (caution!!!)
		# eliminar pods distinto a RUNNING
	#caution!!! for ns in $(oc get namespaces | awk '{print $1}'); do for pod in $(oc get pods --field-selector=status.phase!=Running -n ${ns} | awk '{print $1}'); do oc delete pod --grace-period=1 ${pod} -n ${ns}; done; done;

# SERVICES

	+ get selector
	oc get svc -o json | jq '.items[].spec.selector'
	
	oc get svc -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.selector.app_name}{"\n"}' | grep "b-g"
	
# Build from a template (cuidado hace cambios)
	oc new-app --template=myproject/mytemplate -p=<param_name>=<param_value>

# Create other objects from a file (cuidado hace cambios)
	oc create -f myobject.yaml -n <myproject>
	oc create service clusterip configuration-service --tcp=8080
	oc apply -f dc_micro.yml -n san-geos

# Update an existing object (cuidado hace cambios)
	
	+ SERVICE PATCH
	$ oc patch svc configuration-service --type merge --patch '{"spec":{"ports":[{"port": 8080, "targetPort": 5000 }]}}'
	
	$ oc patch svc b-g-darwin-javase --type merge --patch '{"spec":{"selector":{"app_name": "darwin-javase-11-g","deploymentconfig": "darwin-javase-11"}}}'

	+ ROUTE PATCH
	$ oc patch route $ROUTENAME --type merge --patch '{"spec":{"to": {"kind": "Service", "name": "$BG-SERVICE", "weight": 100}}}'
	

# Acceso ssh running applications: 
	(winpty oc para que acceda a /bin/bash)
	oc exec <pod-name> cat /opt/app-root/myapp.config
	oc exec -ti <pod-name> -- /bin/bash
	oc rsh <pod-name>
	oc debug dc <mydc>
	oc debug --as-root=true dc/front

# Escalar aplicaciones:
	$ oc scale dc <mydc> --replicas=3
	$ oc autoscale dc/app-cli --min 2 --max 5 --cpu-percent=75

# Administrative operations using the adm subcommand:
	Manage user roles:
		$ oc adm policy add-role-to-user admin oia -n python
		$ oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:monitoring:default
		$ oc adm policy add-scc-to-user anyuid -z default

# Mostrar estado nodo/cluster:
		oc adm manage node <node> --schedulable=false
		
# Mostrar ip de los nodo/cluster
		oc get nodes -o wide
				>> https://docs.openshift.com/container-platform/4.1/nodes/nodes/nodes-nodes-viewing.html
		
		oc get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
		oc describe node | egrep "hostname|machineconfig"

# Run cluster diagnostics:
		oc adm diagnostics
		
		oc adm top nodes
		
# Config Maps
		$ oc create configmap pre-krb5conf --from-file=krb5.conf

# Advanced operations (cuidado hace cambios)
	+ Create a config map file and mount it as a volume to a deployment config:
		$ oc create configmap propsfilecm --from-file=application.properties
		$ oc set volumes dc/myapp --add --overwrite=true --name=configmap-volume
 		--mount-path=/data -t configmap --configmap-name=propsfilecm

	+ Create a secret from the CLI and mount it as a volume to a deployment config:
		$ oc create secret generic oia-secret --from-literal=username=myuser
 		--from-literal=password=mypassword
		$ oc set volumes dc/myapp --add --name=secret-volume --mount-path=/opt/app-root/
 		--secret-name=oia-secret

# Logging logs
	oc logs -f bc/openldap
	oc logs backend -c ruby-container
	oc logs dc/config-server --since-time='2022-02-12T09:00:00.00000000Z'
	 --> (https://docs.openshift.com/container-platform/4.5/support/troubleshooting/investigating-pod-issues.html)

	
# Tranfer files rsync
	oc rsync POD:/remote/dir/ ./local/dir
		

# Resource limits:
		oc get dc -o custom-columns=NAME:.metadata.name,RAM_REQ:.spec.template.spec.containers[].resources.requests.memory,RAM_LIMIT:.spec.template.spec.containers[].resources.limits.memory,CPU_REQ:.spec.template.spec.containers[].resources.requests.cpu,CPU_LIMIT:.spec.template.spec.containers[].resources.limits.cpu
		
		oc get dc -o=custom-columns=MICRO:.metadata.name,JAVA_OPTS:'.spec.template.spec.containers[].env[?(@.name == "JAVA_OPTS_EXT")].value'
		
		oc get dc -o=custom-columns=MICRO:.metadata.name,MEMORY_REQUEST:'.spec.template.spec.containers[].resources.requests.memory',MEMORY_LIMIT:'.spec.template.spec.containers[].resources.limits.memory',CPU_REQUEST:'.spec.template.spec.containers[].resources.requests.cpu',CPU_LIMIT:'.spec.template.spec.containers[].resources.limits.cpu',READINESS:'.spec.template.spec.containers[].readinessProbe.httpGet.path',READINESS_TIME:'.spec.template.spec.containers[].readinessProbe.initialDelaySeconds',LIVENESS:'.spec.template.spec.containers[].livenessProbe.httpGet.path',LIVENESS_TIME:'.spec.template.spec.containers[].livenessProbe.initialDelaySeconds'

# TEMPLATES

	# Templates
	oc get templates -n sanes-darwin-catalog-pre 

	# descarga local template (PRE) sanes-darwin-catalog-pre
	oc get template darwin-gateway -n sanes-darwin-catalog-pre -o yaml > darwin-gateway.template.yaml
	oc get template darwin-configuration-service -n sanes-darwin-catalog-pre -o yaml > darwin-configuration-service.template.yaml
	
	# aplicar template con parametros (cuidado hace cambios)
	oc process -f darwin-gateway.template.yaml --param-file parametros/darwin-gateway.params | oc apply -f-
	oc process -f darwin-configuration-service.template.yaml --param-file parametros/darwin-configuration-service.params | oc apply -f-
	
		# path >> parametros/darwin-gateway.params
		APP_NAME=gatewaysist
		GATEWAY_TYPE=sist
		SPRING_PROFILES_ACTIVE=pre	
		DARWIN_REGION=bo1
		
		# path >> parametros/darwin-configuration-service.params
		# darwin-configuration-service.params - ejemplo: cambiar los valores por los correctos
		APP_NAME=changeme
		DARWIN_REGION=changeme
		SPRING_CLOUD_CONFIG_SERVER_GIT_URI=changeme
		SSH_KEY=changeme
		DARWIN_APPKEY=changeme
		DARWIN_SYSTEM=changeme
		DARWIN_SUBSYSTEM=changeme
		DARWIN_APPLICATION=changeme
		DARWIN_SUBAPPLICATION=changeme
	
## LABELS
	
	oc get dc accounts-info -o=custom-columns=MICRO:.metadata.name,LABELS:'.metadata.labels.<labelname>'

## ANNOTATIONS
	
	+ cuidado cambia en todas las rutas
	$ oc annotate route $i --overwrite haproxy.router.openshift.io/disable_cookies=true
	$ oc annotate route $i haproxy.router.openshift.io/disable_cookies-
		
## MISC 

	+ JAVA OPTS
		oc exec -ti <pod> -- /bin/bash
		cat start.d/0.start_javase.sh
		jcmd 1 VM.flags
	+ JAR
	
		# Ver application.yml desde dentro de un POD
		cd /tmp/  ;  mkdir temporal  ;  cd $_  ;  jar xf /tmp/app.jar  ;  cat BOOT-INF/classes/config/application.yml
		
	+ info pods
		oc get dc $DC -o json| jq '.spec.template.spec.containers[].env[]| select(.name=="JAVA_OPTS_EXT")'||jq .value
		oc get service -o wide | grep ^b-g
		oc -o json get dc $DC | jq '.spec.template.spec.containers[].image'
		oc get routes | grep -io '^gate.*\|^dns.*'
		oc get dc -o=custom-columns=NAME:.metadata.name,REPLICAS:.spec.replicas,IMAGEN:'.spec.template.spec.containers[].image'
		oc set env dc --all --list
		
	+ Revisar log level y estado de pods	
		oc get pods -o=custom-columns=NOMBRE:'.metadata.name',LOG_ROOT:'.spec.containers[].env[?(@.name =="ROOT_LOG")].value',\
		TECH_LOG:'.spec.containers[].env[?(@.name =="TECH_LOG")].value',\
		DARWIN_LOGGING:'.spec.containers[].env[?(@.name =="DARWIN_LOGGING")].value',\
		ESTADO_DEL_POD:'status.phase',DARWIN_REGION:'.spec.containers[].env[?(@.name =="DARWIN_REGION")].value'

	+ ENDPOINTS (mostrar fichero JSON)
	
		oc project ; sleep 5 ; oc get dc -o=custom-columns=MICRO:.metadata.name,JAVA_OPTS:'.spec.template.spec.containers[].env[?(@.name == "CONFIG_END_POINT")].value' | grep -i  'green:8080' | grep master
		
	+ get con while
		while read -r line ; do oc project ; sleep 5 ; oc <COMMAND> rc/$line ; done < testRC.txt
	
	+ info cluster elements
		 oc explain dc
		 
	+ Prueba conectividad bbdd
		 curl -kv & dbuser:dbpass@dbhost:port
		 
	+ Carga de ficheros configuration-service, desde un micro se trae la ultima rama
		 curl -v localhost:8080/gatewayweb/pro
		 curl -v config-service:8080/micro-back/pro
		 curl -vv -L http://config-service:8080/micro-front/pro/version-1.0.1/micro-front-pro.yml
		 oc exec -ti <front-pod> -- curl -vv http://config-service:8080/ENVIRO/application.yml | jq
		 oc exec -ti sanope-front-b-65-sll76 -- curl -vv http://config-service:8080/ENVIRO/application.yml | jq
		 
	+ HC (Cuidado patch!!)
		$oc get dc |sed '1d' |awk '{print $1}' |grep -E '.+java.+|api' |xargs -t -I H oc patch dc/H --type=json -p='[{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path", "value":"/actuator/health/readiness"},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/initialDelaySeconds", "value":60},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/failureThreshold", "value":3},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/periodSeconds", "value":10},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/successThreshold", "value":1},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/timeoutSeconds", "value":3},{"op":"replace", "path":"/spec/template/spec/containers/0/livenessProbe/httpGet/path", "value":"/actuator/health/liveness"},{"op":"replace", "path":"/spec/template/spec/containers/0/livenessProbe/initialDelaySeconds", "value":120},{"op":"replace", "path":"/spec/template/spec/containers/0/readinessProbe/failureThreshold", "value":3},{"op":"replace", "path":"/spec/template/spec/containers/0/livenessProbe/periodSeconds", "value":10},{"op":"replace", "path":"/spec/template/spec/containers/0/livenessProbe/successThreshold", "value":1},{"op":"replace", "path":"/spec/template/spec/containers/0/livenessProbe/timeoutSeconds", "value":3}]'

## WORKER

	worker_count=`cat worker/terraform.tfvars | grep worker_count | awk '{print $3}'`
	while [ $(oc get csr | grep worker | grep Approved | wc -l) != $worker_count ]; do
	oc get csr -o json | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
	sleep 3
	done
		
# Variables de entorno OCP

	oc set env dc config-service --list

	oc get dc $DC -o json| jq '.spec.template.spec.containers[].env[]'
	oc get dc $DC -o json| jq '.spec.template.spec.containers[].env[]| select(.name=="$VAR")'||jq .value
	
	VAR="SPRING_CLOUD_CONFIG_SERVER_GIT_URI"
	VAR="DARWIN_LOGGING_KAFKA_SERVER"
	VAR="JAVA_OPTS_EXT"
	VAR="SPRING_APPLICATION_JSON"
	VAR="spring_cloud_config_fail_fast"
	
	APP_NAME=
	SPRING_PROFILES_ACTIVE=
	DARWIN_REGION=
	SPRING_CLOUD_CONFIG_SERVER_GIT_URI=
	SSH_KEY=
	DARWIN_APPKEY=
	DARWIN_SYSTEM=
	DARWIN_SUBSYSTEM=
	DARWIN_APPLICATION=
	DARWIN_SUBAPPLICATION=
	SPRING_CLOUD_CONFIG_SERVER_GIT_REFRESHRATE=31536000
	
##  KEYTAB
	
	## crear
	oc create secret generic keytab-name --from-file=name.keytab

