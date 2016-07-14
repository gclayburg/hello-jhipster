#!groovy
def starttime = System.currentTimeMillis()
stage "provision build node"
node('nodejs4') {  //this node label must match jenkins slave with nodejs installed
println("hi auto again there buddy!! and again. one more time now... again. yo there. hi")
    println("begin: build node ready in ${(System.currentTimeMillis() - starttime) /1000}  seconds")
    wrap([$class: 'TimestamperBuildWrapper']) {  //wrap each Jenkins job console output line with timestamp
        stage "build setup"
        checkout scm
        whereami()
        stage 'production war'
        echo "Building version ${regexVersion()}"

        sh "mvn  -B clean -Pprod verify -DskipTests"
        stage 'docker image/ prod test'
        parallel Java: {
            step([$class: 'ArtifactArchiver',artifacts: '**/target/*.war',fingerprint: true])
            sh "TZ=America/Denver mvn  -B -Pprodtest verify"
            step([$class: 'JUnitResultArchiver',testResults: '**/target/surefire-reports/TEST-*.xml'])
            step([$class: 'JUnitResultArchiver',testResults: '**/target/test-results/karma/TESTS-results.xml',allowEmptyResults: true])
            //name pattern must match path in ./src/test/javascript/karma.conf.js
            println("testing finished in ${(System.currentTimeMillis() - starttime) / 1000} seconds")

        },docker: {
            sh "mvn -B docker:build"
            sh "docker-compose -f src/main/docker/app.yml up -d"
            sh "docker-compose -f src/main/docker/app.yml ps"
            sh "echo \"app is starting... http://\$(docker info | sed -n 's/^Name: //'p):8080/\""
            println("app ready in ${(System.currentTimeMillis() - starttime) / 1000} seconds")
        }

        sh "echo \"Reminder: app should already be running at http://\$(docker info | sed -n 's/^Name: //'p):8080/\""
    }
}

private boolean isMavenProject() {
    echo "looking for a pom.xml"
    try {
        def pomtext = readFile('pom.xml')
        echo "found a pom.xml"
        true
    } catch (ignored){
        false
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
