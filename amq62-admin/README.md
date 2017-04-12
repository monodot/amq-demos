# amq62-admin

Demonstrating how to provide alternative methods of administering JBoss A-MQ when deployed inside OpenShift.

## Changes made

Briefly here are the differences between this build and the standard JBoss A-MQ for OpenShift image.

- Uncomment the Jetty line in `activemq.xml`
- Update Jetty configuration to listen on all addresses (`0.0.0.0`)
