<!-- Copyright 2019 Oath Inc. Licensed under the terms of the Apache 2.0 license. See LICENSE in the project root. -->
# Hosted Vespa sample applications — Basic hosted stateless Vespa application

## The hosted service this application refers to is not yet publicy available — stay tuned

This application demonstrates how to set up and run a basic hosted Vespa application,
and is intended as an introduction to both Vespa and the hosted Vespa service. <!-- TODO LINK -->  

Included are detailed functional tests for the Vespa application, written in JUnit. While the
main intention for such functional tests is to run them in the hosted Vespa continuous
deployment pipeline, <!-- TODO LINK --> to verify changes to the Vespa application before
they are deployed to the production environments, these particular tests also

* demonstrate how to communicate with the Vespa deployment over HTTP,
* serve as introductory documentation for the most central of Vespa's features, and
* provide a starting point for thorough functional tests of your own Vespa application.

The tests require a running Vespa deployment, which is easily obtained in hosted Vespa's
`dev` environment. <!-- TODO LINK --> It is also possible to run the tests against a local
Vespa deployment running with, e.g., docker, as in the other [sample apps](../). 

## Sign up in the hosted Vespa console, and create an application with the wanted name

## Generate developer key pair
Deployments to the `dev` environment require _developer_ access in the Vespa cloud. Navigate to user
management under your tenant, and give yourself the developer role. Then, navigate to key management
for your tenant, generate a key pair if you haven't already got one, and have the public key uploaded.
The private key is your API key, and lets CLIs and test code access the Vespa cloud API with developer privileges.

## Generate application trust certificate and key pair
Deployments in the Vespa cloud are secured by mutual TLS. See
[Data Plane](https://vespa.ai/documentation/security-model.html#data-plane) for more information.
In short, the application trusts clients with the certificates located in `src/main/application/security/clients.pem`.
Generate a certificate to put there, and keep the private key for data plane access, with
```sh
$ openssl req -x509 -nodes -days 14 -newkey ec:<(openssl ecparam -name prime256v1) \
  -keyout $HOME/key.pem -out $HOME/clients.pem
```
Then copy the certificate to `src/main/application/security/`.

## Configure pom.xml for your hosted Vespa application
Set the `tenant`, `application`, `privateKeyFile`, `dataPlaneCertificateFile`, and `dataPlanePrivateKeyFile`
properties in `pom.xml`. The `privateKeyFile` is your API key, while the data plane pair is the one from the above step.

## Deploy to dev and test against it
Command to build and deploy application to the hosted development environment is
<pre>
mvn clean package vespa:deploy 
</pre>
Example System, Staging and Production tests can then be run from an IDE without further setup.
<!-- ... or, add a description for users with older than IntelliJ 2012, and Eclipse ... ??? -->

## Set up a CI job which deploys your application
Command to build and submit application to the hosted Vespa API is
<pre>
mvn clean vespa:compileVersion
mvn -P fat-test-application \
-Dvespaversion="$(cat target/vespa.compile.version)" \
-DauthorEmail=<span style="{background-color: yellow;}">user@domain</span> \
package vespa:submit 
</pre>
To track versions through the pipeline, assuming you're using `git` for version control, you can instead specify
`-Drepository=$(git config --get remote.origin.url) -Dbranch=$(git rev-parse --abbrev-ref HEAD) -Dcommit=$(git rev-parse HEAD) -DauthorEmail=$(git log -1 --format=%aE)`
## Local development

<em>This only works with self-hosted `services.xml` and `hosts.xml`, which can be found in any of the other sample apps.</em>

### Run those JUnit tests against the local docker container
Assuming the below is done, and self-hosted `services.xml` and `hosts.xml` are present in `src/main/application`, put
<pre>
{
  "localEndpoints": {
    "container": "http://localhost:8080/"
  }
}
</pre>
in some file `/path/to/test/config`, and run JUnit tests with `-Dvespa.test.config=/path/to/test/config -Dtest.categories=system
to run all `@SystemTest` classes; `staging` and `production` selects the other types of tests, and `integration` all three.

### Executable example
**Check-out, compile and run:**
<pre data-test="exec">
$ git clone https://github.com/vespa-engine/sample-apps.git
$ export VESPA_SAMPLE_APPS=`pwd`/sample-apps
$ cd $VESPA_SAMPLE_APPS/basic-search-hosted &amp;&amp; mvn clean package
$ docker run --detach --name vespa --hostname vespa-container --privileged \
  --volume $VESPA_SAMPLE_APPS:/vespa-sample-apps --publish 8080:8080 vespaengine/vespa
</pre>

**Wait for the configserver to start:**
<pre data-test="exec" data-test-wait-for="200 OK">
$ docker exec vespa bash -c 'curl -s --head http://localhost:19071/ApplicationStatus'
</pre>

**Deploy the application:**
<pre data-test="exec">
$ docker exec vespa bash -c '/opt/vespa/bin/vespa-deploy prepare /vespa-sample-apps/basic-search-hosted/target/application.zip && \
  /opt/vespa/bin/vespa-deploy activate'
</pre>

**Wait for the application to start:**
<pre data-test="exec" data-test-wait-for="200 OK">
$ curl -s --head http://localhost:8080/ApplicationStatus
</pre>

**Test the application:**
<pre data-test="exec" data-test-assert-contains='"totalCount": 0'>
$ curl -s http://localhost:8080/search/?query=name:foo
</pre>

**Shutdown and remove the container:**
<pre data-test="after">
$ docker rm -f vespa
</pre>
