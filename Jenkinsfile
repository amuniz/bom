properties([disableConcurrentBuilds(abortPrevious: true), buildDiscarder(logRotator(numToKeepStr: '7'))])

def mavenEnv(String sName, Closure body) {
  podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: default
        image: ghcr.io/amuniz/java-build-agent-17:main
        resources:
          requests:
            cpu: 2
            memory: 2Gi
          limits:
            cpu: 2
            memory: 2Gi
        command:
        - sleep
        args:
        - infinity
      - name: jnlp
        resources:
          requests:
            cpu: 1
            memory: 768Mi
          limits:
            cpu: 1
            memory: 768Mi
''') {
    retry(count: 2, conditions: [kubernetesAgent(), nonresumable()]) {
      node(POD_LABEL) {
        container('default') {
          timeout(120) {
            withEnv(["MAVEN_ARGS=-B -Dmaven.repo.local=./m2repo"]) {
              readCache name: sName
              body()
              writeCache includes: 'm2repo/**', name: sName
            }
            if (junit(testResults: '**/target/surefire-reports/TEST-*.xml,**/target/failsafe-reports/TEST-*.xml').failCount > 0) {
              // TODO JENKINS-27092 throw up UNSTABLE status in this case
              error 'Some test failures, not going to continue'
            }
          }
        }
      }
    }
  }
}

@NonCPS
def parsePlugins(plugins) {
  def pluginsByRepository = [:]
  plugins.each { plugin ->
    def splits = plugin.split('\t')
    pluginsByRepository[splits[0].split('/')[1]] = splits[1].split(',')
  }
  pluginsByRepository
}

def pluginsByRepository
def lines

stage('prep') {
  mavenEnv("prep") {
    checkout scm
    withEnv(['SAMPLE_PLUGIN_OPTS=-Dset.changelist -Dignore.dirt=true']) {
      sh '''
      mvn -v
      bash prep.sh
      '''
    }
    dir('target') {
      def plugins = readFile('plugins.txt').split('\n')
      pluginsByRepository = parsePlugins(plugins)

      lines = readFile('lines.txt').split('\n')
    }
    lines.each { line ->
      stash name: line, includes: "pct.sh,excludes.txt,target/pct.jar,target/megawar-${line}.war"
    }
  }
}


branches = [failFast: false]
lines.each {line ->
  if (line != 'weekly') {
    echo "Limited to the weekly line. Skipping '${line}' build."
    return
  }
  pluginsByRepository.each { repository, plugins ->
    branches["pct-$repository-$line"] = {
      mavenEnv("pct-$repository-$line") {
        unstash line
        withEnv([
          "PLUGINS=${plugins.join(',')}",
          "LINE=$line",
          'EXTRA_MAVEN_PROPERTIES=maven.test.failure.ignore=true:surefire.rerunFailingTestsCount=1:maven.repo.local=./m2repo'
        ]) {
          sh '''
          mvn -v
          bash pct.sh
          '''
        }
      }
    }
  }
}
parallel branches
