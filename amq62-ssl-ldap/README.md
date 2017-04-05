# amq62-ssl-ldap

Demonstrates a few security customisations of the [JBoss A-MQ for OpenShift][1] image:

- Configure LDAP authentication for the broker in `activemq.xml`
- Enforce 2-way SSL for client-broker communication
- Enable a 'remote' JMX port on 1099, to allow local administration via the command line `activemq-admin` tool
- Configure LDAP authentication for JMX 
- Configure certificate-based authentication for brokers using `org.apache.activemq.jaas.TextFileCertificateLoginModule`
- Enable only the SSL variants of each protocol

You will need:

- The `ldap` command line tools on your system (e.g. `ldapadd`, `ldapmodify`, etc.)
- A demo LDAP server
- Docker

## Preparation

Create self-signed certificates for 2-way SSL. The broker truststore `broker.ts` must contain the client certificate, and the client truststore `client.ts` must contain the broker certificate:

    $ keytool -genkey -alias broker -keypass changeit -keyalg RSA -keystore broker.ks -dname "CN=broker,O=Wuthering Heights,L=Gimmerton,C=GB" -storepass changeit
    $ keytool -genkey -alias client -keypass changeit -keyalg RSA -keystore client.ks -dname "CN=client,O=Wuthering Heights,L=Gimmerton,C=GB" -storepass changeit

    $ keytool -export -alias broker -keystore broker.ks -file broker_cert -storepass changeit
    $ keytool -import -alias broker -keystore client.ts -file broker_cert -storepass changeit -noprompt

    $ keytool -export -alias client -keystore client.ks -file client_cert -storepass changeit
    $ keytool -import -alias client -keystore broker.ts -file client_cert -storepass changeit -noprompt

Add the broker keystore and truststore into a secret:

    $ oc secrets new amq-app-secret broker.ks broker.ts
    $ oc create sa amq-service-account
    $ oc policy add-role-to-user view -z amq-service-account
    $ oc secrets add sa/amq-service-account secret/amq-app-secret

## Preparation

**Build the image**

Using the [OpenShift source-to-image][s2i] tool:

    $ git clone https://github.com/monodot/ocp-amq-ldap
    $ cd ocp-amq-ldap
    $ s2i build . registry.access.redhat.com/jboss-amq-6/amq62-openshift ocp-amq-ldap --copy
    
**Start and populate a demo LDAP server**

First start a demo LDAP server, port forward so that it's accessible from the host machine [when using Docker for Mac][2]:

    $ docker run -p 389:389 -v /tmp/slapd/data/ldap:/var/lib/ldap \
           -e LDAP_DOMAIN=activemq.apache.org \
           -e LDAP_ORGANISATION="Apache ActiveMQ Test Org" \
           -e LDAP_ADMIN_PASSWORD=sunflower \
           -e LDAP_CONFIG_PASSWORD=sunflower \
           -e CONSOLE_LOG_LEVEL=7 \
           --name openldap \
           -d osixia/openldap:1.1.8

Enable the 'memberOf' attribute in OpenLDAP, so that we can restrict access to JMX by group membership, [solution thanks to this article][3]:

    $ ldapadd -h localhost -p 389 -c -x -D cn=admin,cn=config -w sunflower -f data/memberof-config.ldif

Allow users to search the directory [solution thanks to this GitHub issue][4]:

    $ ldapmodify -h localhost -p 389 -c -x -D cn=admin,cn=config -w sunflower -f data/bind-read.ldif

Now add some test data, provided in `/data` - all passwords are set to _sunflower_:

    $ ldapadd -h localhost -p 389 -c -x -D cn=admin,dc=activemq,dc=apache,dc=org -w sunflower -f data/activemq-openldap.ldif

**(Optional) Some test queries to confirm everything's loaded into OpenLDAP and permissions are working**

As _admin_, see everything in the directory (`objectClass=*`):

    $ ldapsearch -x -h localhost:389 -b "dc=activemq,dc=apache,dc=org" -D "cn=admin,dc=activemq,dc=apache,dc=org" -w sunflower '(objectClass=*)'

As _susan_, see which groups _susan_ is a member of (`memberOf`):

    $ ldapsearch -x -h localhost:389 -b "dc=activemq,dc=apache,dc=org" -D "uid=susan,ou=User,ou=ActiveMQ,dc=activemq,dc=apache,dc=org" -w sunflower '(uid=susan)' memberOf

## Option A: Run AMQ in a container using Docker

Start a new container from the LDAP-configured A-MQ image, and point it to the _openldap_ LDAP server:

    $ OPENLDAP_IP=`docker inspect -f '{{ .NetworkSettings.IPAddress }}' openldap`
    $ docker run -d -e LDAP_HOST=$OPENLDAP_IP -e LDAP_USER=cn=mqbroker,ou=Services,dc=activemq,dc=apache,dc=org -e LDAP_PASSWORD=sunflower --name ocp-amq-ldap ocp-amq-ldap

To send a test message as the LDAP user `jdoe`, start a bash shell in the container and run:

    $ docker exec -it ocp-amq-ldap /opt/amq/bin/activemq producer --messageCount 1 --user jdoe --password sunflower

## Option B: Run in a local OpenShift cluster

First, I start up my demo LDAP server using the instructions above. I also run the `s2i build` to build the Docker image locally.

Then I start my Nexus container, and port forward on port 8123 (I cannot address the container by its IP address when using Docker for Mac [because of this lil guy][2]), tag and push the image there:

(Note that I have already configured my Docker registry on port 8123, using the Nexus admin console)

    $ mkdir -p ~/volumes/nexus/data
    $ chown -R 200 ~/volumes/nexus/data
    $ docker run -d -p 8083:8081 -p 8123:8123 --name nexus -v ~/volumes/nexus/data:/nexus-data sonatype/nexus3

Wait for Nexus to start, then log in to the registry and push the image:

    $ docker login -u admin -p admin123 localhost:8123
    $ docker tag ocp-amq-ldap localhost:8123/foo/ocp-amq-ldap:1
    $ docker push localhost:8123/foo/ocp-amq-ldap:1

Create a project and the necessary stuff in OpenShift to run the image:

    $ oc new-project amqdemo
    $ export NEXUS_IP=`docker inspect --format '{{ .NetworkSettings.IPAddress }}' nexus`

    $ oc secrets new-dockercfg my-secret \
        --docker-server=$NEXUS_IP:8123 --docker-username=admin \
        --docker-password=admin123 --docker-email=a@a.com

    $ oc secrets add serviceaccount/default secrets/my-secret --for=pull

    $ OPENLDAP_IP=`docker inspect -f '{{ .NetworkSettings.IPAddress }}' openldap`

Now run the image, injecting the LDAP auth details as environment variables, and setting the imagePullPolicy to Always (you will need `jq` installed for this command):

    $ oc run ocp-amq-ldap \
        --env=LDAP_HOST=$OPENLDAP_IP,LDAP_PASSWORD=sunflower \
        --image=$NEXUS_IP:8123/foo/ocp-amq-ldap:1 \
        --dry-run -o json \
        | jq '.spec.template.spec.containers[0].env |= (. + [{"name": "LDAP_USER", "value": "cn=mqbroker,ou=Services,dc=activemq,dc=apache,dc=org"}])' \
        | jq '.spec.template.spec.containers[0].imagePullPolicy = "Always"' \
        | oc create -f -

Go into the pod, and use the `activemq-admin` utility to administrate the broker:

    $ oc rsh <your-pod-name>
    $ export ACTIVEMQ_OPTS="-Dactivemq.jmx.user=susan -Dactivemq.jmx.password=sunflower"
    $ /opt/amq/bin/activemq-admin dstat

## Conclusion

LDIF is a massive faff. 😷

[1]: https://access.redhat.com/documentation/en/red-hat-jboss-middleware-for-openshift/3/paged/red-hat-jboss-a-mq-for-openshift/
[2]: https://docs.docker.com/docker-for-mac/networking/#per-container-ip-addressing-is-not-possible
[3]: http://www.adimian.com/blog/2014/10/how-to-enable-memberof-using-openldap/
[4]: https://github.com/osixia/docker-openldap/issues/82
[s2i]: https://github.com/openshift/source-to-image



