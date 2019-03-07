= aws-modernization-workshop
Workshop for modernization

== Local Testing
To test any changes you have made locally make sure you have docker installed. Make sure you are logged into the correct AWS account.

=== Run the following commands.
.docker login
[source,shell]
----
eval `aws ecr get-login --no-include-email --region us-east-1`
----

.docker image pull
[source,shell]
----
docker pull 755152575036.dkr.ecr.us-east-1.amazonaws.com/aws-workshopper:latest
----

Make sure you are in this repository root.

.docker run
[source,shell]
----
docker run --rm -ti \
           -p 8080:8080 \
           -v `pwd`:/content \
           -e CONTENT_URL_PREFIX=file:///content \
           -e WORKSHOPS_URLS=file:///content/workshops/modernization-august-2018.yml \
           755152575036.dkr.ecr.us-east-1.amazonaws.com/aws-workshopper:latest
----

You should now be able to see the workshop on http://localhost:8080
