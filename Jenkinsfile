#!groovy
node('nodejs4.4.5') {  //this node label must match jenkins slave with nodejs installed
    println("begin build")
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
        stage 'maven build/test, gulp test'
        // run the build a little faster: maven in one thread, gulp test in another
        parallel Java: {
            echo "in java branch"
            sh "docker ps"
            sh "docker info"
            sh "mvn -B -Pprod clean verify -DskipTests" // just the war, thank you very much
            sh "mvn -B -Pprod package docker:build"  //stuff into docker image
            echo "lets do docker-compose"
            sh "docker-compose -f src/main/docker/app.yml up -d"
            step([$class: 'ArtifactArchiver',artifacts: '**/target/*.war',fingerprint: true])
            step([$class: 'JUnitResultArchiver',testResults: '**/target/surefire-reports/TEST-*.xml'])
        },JS: {
            sh "npm install"  //nodejs, npm, and gulp must be installed in Jenkins slave Docker container
            sh "gulp test"
            step([$class: 'JUnitResultArchiver',testResults: '**/target/test-results/karma/TESTS-results.xml', allowEmptyResults: true]) //name pattern must match path in ./src/test/javascript/karma.conf.js
        }
        stage 'archive test results/war'
    }
}

private void tooloverride() {
    try {
        def javaHOME = tool 'Oracle JDK 8u25' //tool name must match JDK installation in Jenkins->Configuration
        echo "tool override: using Java from Jenkins->configuration->JDK installations"
        env.PATH = "${javaHOME}/bin:${env.PATH}"
    } catch (Exception e) {
        echo "no java tool installed"
    }

    try{
        def mvnHome = tool 'M3' //tool name must match maven installation in Jenkins->Configuration
        echo "tool override: using Maven from Jenkins->configuration->Maven installations"
        env.PATH = "${mvnHome}/bin:${env.PATH}"
    } catch (Exception e){
        echo "no maven tool installed"
    }
    whereami()
}

private void whereami() {
    echo "Build is running with these settings:"
    sh "pwd"
    sh "ls -la"
    sh "echo path is \$PATH"
    sh """
uname -a
java -version
mvn -v
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
