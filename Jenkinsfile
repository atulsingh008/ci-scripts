#!/usr/bin/env groovy
// -*- mode: groovy; tab-width: 2; groovy-indent-offset: 2 -*-
// Copyright (c) 2017 Wind River Systems Inc.
//
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in all
// copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
// SOFTWARE.

node('docker') {

  // Node name is from docker swarm is hostname + dash + random string. Remove random part of recover hostname
  def hostname = "${NODE_NAME}"
  hostname = hostname[0..-10]
  def common_docker_params = "--name build-${BUILD_ID} --hostname ${hostname} -t --tmpfs /tmp --tmpfs /var/tmp -v /etc/localtime:/etc/localtime:ro -u 1000"

  stage('Docker Run Check') {
    dir('ci-scripts') {
      git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
    }
    sh "${WORKSPACE}/ci-scripts/docker_run_check.sh"
  }
  stage('Cache Sources') {
    dir('ci-scripts') {
      git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
    }
    docker.withRegistry('http://${REGISTRY}') {
      docker.image("${IMAGE}").inside(common_docker_params) {
        withEnv(['LANG=en_US.UTF-8', "BASE=${WORKSPACE}", "REMOTE=${REMOTE}"]) {
          sh "${WORKSPACE}/ci-scripts/wrlinux_update.sh ${BRANCH}"
        }
      }
    }
  }

  try {
    stage('Layerindex Setup') {
      // if devbuilds are enabled, start build in same network as layerindex
      if (params.DEVBUILD_ARGS != "") {
        dir('ci-scripts') {
          git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
        }
        devbuild_args = "${DEVBUILD_ARGS}".tokenize(',')
        withEnv(devbuild_args) {
          dir('ci-scripts/layerindex') {
            sh "./layerindex_start.sh"
            sh "./layerindex_layer_update.sh"
          }
        }
      }
      else {
        println("Not starting local LayerIndex")
      }
    }

    stage('Build') {
      dir('ci-scripts') {
        git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
      }

      def docker_params = common_docker_params
      if (params.TOASTER == "enable") {
        docker_params = docker_params + ' --expose=8800 -P -e "SERVICE_NAME=toaster" -e "SERVICE_CHECK_HTTP=/health"'
      }

      def env_args = ['LANG=en_US.UTF-8', "MESOS_TASK_ID=${BUILD_ID}", "BASE=${WORKSPACE}"]

      if (params.DEVBUILD_ARGS != "") {
        devbuild_args = "${DEVBUILD_ARGS}".tokenize(',')
        docker_params = docker_params + ' --network=build${BUILD_ID}_default'
        env_args = env_args + devbuild_args
      }
      docker.withRegistry('http://${REGISTRY}') {
        docker.image("${IMAGE}").inside(docker_params) {
          withEnv(env_args) {
            sh "mkdir -p ${WORKSPACE}/builds"
            sh "${WORKSPACE}/ci-scripts/jenkins_build.sh"
          }
        }
      }
    }
  }
  finally {
    stage('Layerindex Cleanup') {
      if (params.DEVBUILD_ARGS != "") {
        dir('ci-scripts') {
          git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
        }
        dir('ci-scripts/layerindex') {
          sh "./layerindex_stop.sh"
        }
      }
      else {
        println("No LayerIndex Cleanup necessary")
      }
    }

    stage('Post Process') {
      dir('ci-scripts') {
        git(url:'git://ala-git.wrs.com/projects/wrlinux-ci/ci-scripts.git', branch:"${CI_BRANCH}")
      }
      docker.withRegistry('http://${REGISTRY}') {
        def postprocess_args = "${POSTPROCESS_ARGS}".tokenize(',')
        // hard code network so postbuild container can access internal rsyncd server using DNS
        def docker_params = common_docker_params + ' --network ciscripts_ci_net'
        docker.image("${POSTPROCESS_IMAGE}").inside(docker_params) {
          withEnv(postprocess_args) {
            sh "${WORKSPACE}/ci-scripts/build_postprocess.sh"
          }
        }
      }
    }
  }
}
