## ENVIRONMENT
export DEV_PROJECT=fruit-service-dev
export TEST_PROJECT=fruit-service-test

## CLEAN BEFORE RUNNING THE DEMO
oc delete project ${DEV_PROJECT}
oc delete project ${TEST_PROJECT}

## [OPTIONAL] ADDITIONAL DEPLOYMENT TYPES

### DEPLOY FROM GIT REPO

1. Log in as 'developer' in OCP web console.
2. Change to DEVELOPER view and create project ${DEV_PROJECT}
3. Add -> Database
3.1 Uncheck Type->Operator Backed
3.2 PostgreSQL (Non-ephemeral) then click on Instantiate Template
3.3 Database Service Name: my-database
    PostgreSQL Connection Username: luke
    PostgreSQL Connection Password: secret
    PostgreSQL Database Name: my_data
    CLICK on Create
3.4 Add -> From Git
    Git Repo URL: https://github.com/cvicens/spring-boot-fruit-service
    Java 8
    General Application: fruit-service-app 
    Name: fruit-service-git
    Check Deployment <===
    Click on Deployment to add env variables
    - DB_USERNAME from secret my-database...
    - DB_PASSWORD 
    - JAVA_OPTIONS: -Dspring.profiles.active=openshift
    Click on labels
    - app=fruit-service-git version=1.0.0

### DECORATE DATABASE AND GIT DEPLOYMENT

oc label deployment/fruit-service-git app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate deployment/fruit-service-git app.openshift.io/connects-to=my-database --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT}

### DEPLOY JENKINS HERE TO SAVE TIME... 

Details bellow in section ### DEPLOY JENKINS

### DEPLOY WITH F8

oc project ${DEV_PROJECT}
mvn clean fabric8:deploy -DskipTests -Popenshift
oc label dc/fruit-service-dev app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/fruit-service-dev app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate dc/fruit-service-dev app.openshift.io/connects-to=my-database --overwrite=true -n ${DEV_PROJECT} 

## COMPLETE PIPELINE

### PROJECTS
oc new-project ${TEST_PROJECT}
oc new-project ${DEV_PROJECT}

### DEPLOY JENKINS
#oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins:4.2.5 -n ${DEV_PROJECT}
oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins:2 -n ${DEV_PROJECT}
oc label dc/jenkins app.openshift.io/runtime=jenkins --overwrite=true -n ${DEV_PROJECT} 

### DATABASES [SKIP IN DEV_PROJECT IF DONE BEFORE]
oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${DEV_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${DEV_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT} 

oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${TEST_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${TEST_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${TEST_PROJECT} 

### SECURITY/ROLES
oc policy add-role-to-user edit system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user view system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user system:image-puller system:serviceaccount:${TEST_PROJECT}:default -n ${DEV_PROJECT}

### CREATE PIPELINE
oc apply -n ${DEV_PROJECT} -f jenkins-pipeline-complex.yaml

### START PIPELINE
oc start-build bc/fruit-service-pipeline-complex -n ${DEV_PROJECT}

===================== THE END =====================


==> Cambiar pom.xml -> JDK version

git commit -a -m "jdk 8:1.5"
git push origin master

oc start-build bc/fruit-service-pipeline-complex -n ${DEV_PROJECT}

==> Probar jdk version en DEV antes de aprobar!

==> Aprobar y probar jdk version en TEST
