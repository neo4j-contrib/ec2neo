NEO4J on AWS EC2
----------------

CloudFormation templates that bootstrap Neo4j onto an Amazon EC2 machine, in Amazon Linux or Ubuntu flavours.

[Documentation](/README.CLOUDFORMATION.md)

Caveats:
* They use the OpenJDK JVM, not Oracle,
* There's no backup (but they do write to an Amazon EBS volume)
* It's Neo4j Community, not Enterprise.
