#!groovy
def starttime = System.currentTimeMillis()
node('nodejs4.4.5') {  //this node label must match jenkins slave with nodejs installed
    println("begin: build node ready in ${(System.currentTimeMillis() - starttime) /1000}  seconds")
    wrap([$class: 'TimestamperBuildWrapper']) {  //wrap each Jenkins job console output line with timestamp
        stage "build setup"
        checkout scm
        def ver = regexVersion()
        if (ver) {
            echo "Building version $ver"
        } else {
            echo "Building version unknown"
        }
        tooloverride()
        whereami()
        stage 'production war'
        sh "mvn  -B clean -Pprod verify -DskipTests"
        stage 'docker image/prod test'
        parallel Java: {
            step([$class: 'ArtifactArchiver',artifacts: '**/target/*.war',fingerprint: true])
            sh "mvn  -B -Pprodtest verify"
            step([$class: 'JUnitResultArchiver',testResults: '**/target/surefire-reports/TEST-*.xml'])
            step([$class: 'JUnitResultArchiver',testResults: '**/target/test-results/karma/TESTS-results.xml',allowEmptyResults: true])        //name pattern must match path in ./src/test/javascript/karma.conf.js
            println("testing finished in ${(System.currentTimeMillis() -starttime) /1000} seconds")

        }, docker: {
            sh "mvn -B docker:build"
            sh "docker-compose -f src/main/docker/app.yml up -d"
            sh "docker-compose -f src/main/docker/app.yml ps"
            sh "echo \"app should be running at http://\$(docker info | sed -n 's/^Name: //'p):8080/\""
            println("app ready in ${(System.currentTimeMillis() -starttime) /1000} seconds")
        }
    }
}

private void runSingleThreadBuild() {

    sh "mvn -B -Pprod verify docker:build" // run tests
    stage 'docker build'
    echo "NA"
//        sh "mvn docker:build"  //docker-maven-plugin builds our docker image
    stage 'docker up'
    sh "docker-compose -f src/main/docker/app.yml up -d"
    sh "docker-compose -f src/main/docker/app.yml ps"
}

private void runParallel() {
    // run the build a little faster: maven in one thread, gulp test in another
    parallel Java: {
        echo "in java branch"
        sh "mvn -B -Pprod clean verify -DskipTests" // just the war, thank you very much
        parallel JavaPackage: {  //create java war and docker image in one thread, Java testing via maven in another
            step([$class: 'ArtifactArchiver',artifacts: '**/target/*.war',fingerprint: true])
            sh "mvn docker:build"  //docker-maven-plugin builds our docker image
        },JavaTest: {
            sh "mvn -B -Pprod verify" // run tests
            step([$class: 'JUnitResultArchiver',testResults: '**/target/surefire-reports/TEST-*.xml'])
        }
    },JS: {
        sh "npm install"  //nodejs, npm, and gulp must be installed in Jenkins slave Docker container
        sh "gulp test"
        step([$class: 'JUnitResultArchiver',testResults: '**/target/test-results/karma/TESTS-results.xml',allowEmptyResults: true])
        //name pattern must match path in ./src/test/javascript/karma.conf.js
    }
}

private void tooloverride() {
    try {
        def javaHOME = tool 'Oracle JDK 8u25' //tool name must match JDK installation in Jenkins->Configuration
        echo "tool override: using Java from Jenkins->configuration->JDK installations"
        env.PATH = "${javaHOME}/bin:${env.PATH}"
    } catch (Exception e) {
    }

    try{
        def mvnHome = tool 'M3' //tool name must match maven installation in Jenkins->Configuration
        echo "tool override: using Maven from Jenkins->configuration->Maven installations"
        env.PATH = "${mvnHome}/bin:${env.PATH}"
    } catch (Exception e){
    }
}

private void whereami() {
    /**
     * Runs a bunch of tools that we assume are installed on this node
     */
    echo "Build is running with these settings:"
    sh "pwd"
    sh "ls -la"
    sh "echo path is \$PATH"
    sh """
uname -a
java -version
mvn -v
docker ps
docker info
docker-compose -f src/main/docker/app.yml ps
docker-compose version
npm version
gulp --version
bower --version
"""
}

//def pomVersion(){
    //more accurate way to parse version # from pom.xml, but Jenkins pipeline generates many security errors
//    def pomtext = readFile('pom.xml')
//    def pomx = new XmlParser().parseText(pomtext)
//    pomx.version.text()
//}

def regexVersion(){
    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
    matcher ? matcher[1][1] : null   //blindly assume the 1st version occurence is the parent, and 2nd is our project
}
