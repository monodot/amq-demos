# amq62-basic-s2i

Demonstrates how to customise the JBoss A-MQ for OpenShift image, by extending the image using S2I.

The [JBoss A-MQ for OpenShift template][1] normally just deploys the A-MQ image with the environment variables you choose. However, we can also use the S2I feature in OpenShift to effectively customise this image any way we like.

First we create an `.s2i` directory in a Git repo to store our custom S2I build scripts. Then we add our custom configuration, and create a `.s2i/bin/assemble` script which copies our custom file(s) to the relevant locations inside the image.

When a new app is created in OpenShift from this repository, OpenShift will detect the S2I build and use the provided scripts to build a new image and then deploy it into the environment.

## Usage

To see this in action:

1. Fork this repo into your own GitHub account so that you can make changes.

2. Make any changes you wish to the `configuration/openshift-activemq.xml` file.

3. Create the template in OpenShift using the provided JSON file - `oc create -f amq62-basic-s2i.json`

4. From OpenShift Console, create a new app from the template. A build will start and a new A-MQ broker will be deployed using your custom image.

(Optional) To trigger a new build and deployment when changes are pushed to the Git repo:

1. Extract the GitHub webhook secret URL from the BuildConfig:

        $ oc describe bc <name>

2. Copy the GitHub webhook URL.

3. In your GitHub repo, go to Settings &rarr; Webhooks &rarr; Add Webhook. Add a new webhook with the URL, setting the type to `application/json`. Make sure to disable SSL validation if using self-signed certificates.

[1]: https://github.com/jboss-openshift/application-templates/blob/master/amq/amq62-basic.json
