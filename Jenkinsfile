#!groovy

/*
 * Copyright © 2017, 2018 IBM Corp. All rights reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file
 * except in compliance with the License. You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed under the
 * License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
 * either express or implied. See the License for the specific language governing permissions
 * and limitations under the License.
 */

def getEnvForDest(dest, auth) {
    // Define the matrix environments
    CLOUDANT_ENV = ['DB_HTTP=https', 'DB_HOST=clientlibs-test.cloudant.com', 'DB_PORT=443']
    CONTAINER_ENV = ['DB_HTTP=http', 'DB_PORT=5984']
    def testEnvVars =[]
    if (dest == 'cloudant') {
       testEnvVars.addAll(CLOUDANT_ENV)
       testEnvVars.add('RUN_CLOUDANT_TESTS=1')
       testEnvVars.add('SKIP_DB_UPDATES=1')
    } else {
       testEnvVars.addAll(CONTAINER_ENV)
       switch(dest) {
           case ~/couchdb:.*/:
               testEnvVars.add('DB_PORT=5984')
               break
           case 'ibmcom/cloudant-developer':
               testEnvVars.add('DB_PORT=80')
               testEnvVars.add('RUN_CLOUDANT_TESTS=1')
               testEnvVars.add('SKIP_DB_UPDATES=1')
               break
           default:
               error("Unknown test env")
       }
    }

    switch(auth) {
      case 'basic':
        testEnvVars.add("RUN_BASIC_AUTH_TESTS=1")
        break
      case 'cookie':
        break
      case 'iam':
        // Setting IAM_API_KEY forces tests to run using an IAM enabled client.
        testEnvVars.add("IAM_API_KEY=$DB_IAM_API_KEY")
        break
      default:
        error("Unknown test suite environment ${auth}")
    }
    return testEnvVars
}

def test_python(name, dest, auth) {
  // Define the test routine for different python versions
  def testRun = {
    sh "wget -S --spider --retry-connrefused ${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}; done"
    switch(dest) {
      case ~/couchdb:latest/:
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_replicator", authentication: env.CREDS_ID
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_users", authentication: env.CREDS_ID
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_global_changes", authentication: env.CREDS_ID
        break
      case 'ibmcom/cloudant-developer':
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_replicator", authentication: env.CREDS_ID
        break
      default:
        break
    }
    try {
      // Unstash the source in this image
      unstash name: 'source'
      if (env.DB_USER && dest == 'ibmcom/cloudant-developer') {
        sh """export DB_USER=${env.DB_USER}
              export DB_PASSWORD=${env.DB_PASSWORD}"""
      }
      sh """pip install -r requirements.txt
            pip install -r test-requirements.txt
            pylint ./src/cloudant
            export DB_URL=${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}
            nosetests -w ./tests/unit --with-xunit"""
    } finally {
      // Load the test results
      junit 'nosetests.xml'
    }
  }
  def runInfo = [imageName: name, envVars: getEnvForDest(dest, auth), runArgs: '-u 0']
  // Add test suite specific environment variables
  if (dest == 'cloudant') {
    runInfo.envVars.add('CREDS_ID=clientlibs-test')
    runInfo.creds = [usernamePassword(credentialsId: 'clientlibs-test', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
    runIn.dockerEnv(runInfo, testRun)
  } else {
    // Use the sidecar as the test target host
    runInfo.envVars.add('DB_HOST=$SIDECAR_0')
    // Add credentials for the cloudantdeveloper image
    if (dest == 'ibmcom/cloudant-developer') {
      runInfo.envVars.add('CREDS_ID=cloudant-developer')
      runInfo.creds = [usernamePassword(credentialsId: 'cloudant-developer', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
    }
    runIn.dockerEnvWithSidecars(runInfo, [[imageName: dest]], testRun)
  }
}

// Start of build
stage('Checkout'){
  // Checkout and stash the source
  node{
    checkout scm
    stash name: 'source'
  }
}
stage('Test'){
  // Run tests in parallel for multiple python versions
  def testAxes = [:]
  ['2.7', '3.6'].each { v ->
    ['couchdb:1.7.1', 'couchdb:latest', 'cloudant'].each { c ->
      ['basic','cookie','iam'].each { auth ->
        if(auth != 'iam' && c != ~/couchdb:.*/) {
          testAxes.put("Python${v}_${c}", {test_python("python:${v}", c, auth)})
        }
      }
    }
  }
  parallel(testAxes)
}