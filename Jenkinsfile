#!/usr/bin/env groovy
@Library("global-pipeline-libraries@v1.9.1") _

void pushBranchAndTags(String branchName) {
  withCredentials([usernamePassword(credentialsId: 'hurley-github2', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
    sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mike-hurley-rally/hurley-versioning.git ${branchName} --tags"
  }
}

pipeline {
  agent {
    label 'ec2'
  }
  parameters {
  /*parameters {
        choice(name: 'release', choices: ['rally-versioning', 'patch', 'minor', 'major'], description: 'Semantic version increment')
    }*/
    string(name: 'gitRef', trim: true, description: 'Git ref to start the release from. Git sha for a new release. Optional for hotfixes on release branches.')
    choice(name: 'releaseType', choices: ['minor', 'major'], description: 'Semantic version increment. Ignored if the git ref is a release branch.')
    booleanParam(name: 'isNewRelease', defaultValue: false, description: 'NOT FOR HUMAN USE. Tells the release branch this is the initial release of the branch.')
  }
  stages {
    stage('Validate inputs') {
      steps {
        script {
          def onRB = env.GIT_BRANCH.startsWith('RB-')
          if (params.isNewRelease && !onRB) error 'Parameter isNewRelease should be set only on release branches.'
          if (!onRB && (params.gitRef == null || params.gitRef.trim().isEmpty())) error 'Parameter gitRef must be a git sha.'
          if (!onRB && (params.releaseType == null || params.releaseType.trim().isEmpty())) error 'Parameter releaseType is required.'
          if (params.gitRef.length() == 40) {
            if (onRB) error 'Cannot start a new release from a release branch.'
            if (!params.gitRef.matches('[0-9a-fA-F]{40}')) error 'Parameter gitRef must be a git sha.'
          } else if (params.gitRef.startsWith('RB-')) {
            if (!onRB) error 'Start hotfix builds from the relevant release branch.'
          }
          else error 'Parameter gitRef must be a git sha.'
        }
      }
    }
    stage('Start new release from master') {
      when {
        expression { params.gitRef.length() == 40 }
      }
      steps {
        script {
          def version = rally_git_nextTag(params.releaseType).replaceFirst('v', '')
          def (major, minor, patch) = version.tokenize('.')
          def rbName = "RB-${major}.${minor}"
          sh "git tag v${version} ${params.gitRef}"
          sh "git branch ${rbName} ${params.gitRef}"
          pushBranchAndTags(rbName)
          build job: "..", propagate: true, wait: true, quietPeriod: 0
          build job: rbName, propagate: false, wait: false, quietPeriod: 0, parameters: [
                  string(name: 'gitRef', value: rbName),
                  booleanParam(name: 'isNewRelease', value: true)
          ]
        }
        // calc version
        // make major+minor rb at sha
        // tag sha with version
        // force jenkins branch scan
        // kick off rb build
      }
    }
    stage('Continue new release on the RB') {
      when {
        allOf {
          branch 'RB-*'
          expression { params.isNewRelease }
        }
      }
      steps {
        script {
          String[] tags = (sh(script: 'git tag', returnStdout: true)).tokenize('\n')
          for(int i = 0; i < tags.length; i++) {
            if (tags[i].startsWith('v') && tags[i].contains('.')) {
              env.VERSION = tags[i].replaceFirst('v', '')
              break
            }
          }
        }
      }
    }
    stage('Start new hotfix from RB') {
      when {
        allOf {
          branch 'RB-*'
          expression { !params.isNewRelease }
        }
      }
      steps {
        script {
          def version = rally_git_nextTag('patch').replaceFirst('v', '')
          env.VERSION = version
          sh "git tag v${version}"
          pushBranchAndTags(env.GIT_BRANCH)
        }

        // calc patch version
        // tag tip with version
      }
    }
    stage('DO THE RELEASE') {
      steps {
        echo "Releasing ${env.VERSION}!!"
      }
    }
  }
}