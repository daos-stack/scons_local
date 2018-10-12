#!/usr/bin/env groovy
// Copyright (c) 2018 Intel Corporation
//
// Permission is hereby granted, free of charge, to any person obtaining a
// copy of this software and associated documentation files (the "Software"),
// to deal in the Software without restriction, including without limitation
// the rights to use, copy, modify, merge, publish, distribute, sublicense,
// and/or sell copies of the Software, and to permit persons to whom the
// Software is furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
// THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
// FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
// DEALINGS IN THE SOFTWARE.

@Library(value="pipeline-lib@jemalmbe_test_a") _

pipeline {
  agent any

  environment {
    GITHUB_CREDS = 'aa4ae90b-b992-4fb6-b33b-236a53a26f77'
    BAHTTPS_PROXY = "${env.HTTP_PROXY ? '--build-arg HTTP_PROXY="' + env.HTTP_PROXY + '" --build-arg http_proxy="' + env.HTTP_PROXY + '"' : ''}"
    BAHTTP_PROXY = "${env.HTTP_PROXY ? '--build-arg HTTPS_PROXY="' + env.HTTPS_PROXY + '" --build-arg https_proxy="' + env.HTTPS_PROXY + '"' : ''}"
    UID=sh(script: "id -u", returnStdout: true)
    BUILDARGS = "--build-arg NOBUILD=1 --build-arg UID=$env.UID $env.BAHTTP_PROXY $env.BAHTTPS_PROXY"
    HWLOC_COMMIT = 'refs/tags/hwloc-1.11.5'
    ISAL_COMMIT = 'refs/tags/v2.24.0'
    MERCURY_COMMIT = '674e7f2bd17b5d8b85606cd152dd1bc189899b0e'
    OFI_COMMIT = '8c33f9d63d536cc3781017dd25b7bb480ac96cb5'
    OMPI_COMMIT = '373098d8ae0053af85cc1d49a58dbe933a31a50a'
    OPENPA_COMMIT = '8e1e74feb22d2e733f34a96e6c7834fed3073c52'
    PMIX_COMMIT = '8c2ffe7e6837a2fb76b882e4c5765032b2b84fa9'
  }

  options {
    // preserve stashes so that jobs can be started at the test stage
    preserveStashes(buildCount: 5)
    checkoutToSubdirectory('scons_local')
  }

  stages {
    stage('Phase 1') {
    parallel {
      stage ('code_lint_check') {

        // Just use CentOS for lint checks.
        agent {
           dockerfile {
             filename 'Dockerfile.centos.7'
             dir 'scons_local/docker'
             label 'docker_runner'
             additionalBuildArgs '$BUILDARGS'
           }
        } // agent
        steps {
          checkPatch review_creds: "${env.GITHUB_CREDS}"
        } // steps
      } // stage ('code_lint_check')
      stage ('CentOS7 openpa prebuild') {
        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {
          // Older scons_local_review only used master branch.
          // Newer one looked up the last known good build of master branch.
          // Pipeline currently does not have access that info.
          // sh "rm -rf openpa"
          // checkoutScm url: 'http://git.mcs.anl.gov/radix/openpa.git',
          //            checkoutDir: 'openpa',
          //            branch: "${env.OPENPA_COMMIT}",
          //            cleanAfterCheckout: true

          sh 'mkdir -p testbin; rm -rf testbin/*'

          sconsBuild target: 'openpa',
                     directory: 'scons_local',
                     scm: [url: 'http://git.mcs.anl.gov/radix/openpa.git',
                           branch: "${env.OPENPA_COMMIT}",
                           cleanAfterCheckout: true],
                     no_install: true,  // No separate install step
                     TARGET_PREFIX: '/testbin',
                     target_work: 'testbin'
 
          /*sh('''TARGET='openpa'
                rm -rf /testbin/${TARGET}
                rm -rf testbin
                mkdir -p testbin/${TARGET}
                ln -s ${PWD}/testbin/${TARGET} /testbin/${TARGET}
                SCONS_ARGS='TARGET_PREFIX=/testbin'
                SCONS_ARGS+=' --config=force --build-deps=yes'
                SCONS_ARGS+=" SRC_PREFIX=${PWD}/${TARGET}"
                SCONS_ARGS+=" REQUIRES=${TARGET}"
                pushd scons_local
                  scons -c
                  rm -rf _build.external install build
                  if ! scons $SCONS_ARGS; then
                    rc=\${PIPESTATUS[0]}
                    echo "${SCONS_ARGS} failed"
                    cat config.log || true
                    exit \$rc
                  fi
                popd''')
          */
          echo "openpa build succeeded"
          stash name: 'CentOS7-openpa', includes: 'testbin/openpa/**'
        } // steps
      } // stage ('CentOS7 openpa prebuild')
      stage ('CentOS7 hwloc prebuild') {
        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {

          sconsBuild target: 'hwloc',
                     directory: 'scons_local',
                     scm: [url: 'https://github.com/open-mpi/hwloc.git',
                           branch: "${env.HWLOC_COMMIT}",
                           cleanAfterCheckout: true],
                     no_install: true,  // No separate install step
                     TARGET_PREFIX: '/testbin',
                     target_work: 'testbin',
                     prebuild: 'pushd ${WORKSPACE}/hwloc;' +
                               ' ./autogen.sh; popd'
          
          /*
          checkoutScm url: 'https://github.com/open-mpi/hwloc.git',
                      branch: "${env.HWLOC_COMMIT}",
                      checkoutDir: 'hwloc',
                      cleanAfterCheckout: true
          sh('''TARGET='hwloc'
                # because we are using git instead of curl
                pushd hwloc
                  ./autogen.sh
                popd
                rm -rf /testbin/${TARGET}
                rm -rf testbin
                mkdir -p testbin/${TARGET}
                ln -s ${PWD}/testbin/${TARGET} /testbin/${TARGET}
                SCONS_ARGS='TARGET_PREFIX=/testbin'
                SCONS_ARGS+=' --config=force --build-deps=yes'
                SCONS_ARGS+=" SRC_PREFIX=${PWD}/${TARGET}"
                SCONS_ARGS+=" REQUIRES=${TARGET}"
                pushd scons_local
                  scons -c
                  rm -rf _build.external install build
                  if ! scons $SCONS_ARGS; then
                    rc=\${PIPESTATUS[0]}
                    echo "${SCONS_ARGS} failed"
                    cat config.log || true
                    exit \$rc
                  fi
                popd''')
          */
          echo "hwloc build succeeded"
          stash name: 'CentOS7-hwloc', includes: 'testbin/hwloc/**'
        } // steps
      } // stage ('CentOS7 hwloc prebuild')
      stage ('CentOS7 isal prebuild') {
        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {
          sconsBuild target: 'isal',
                     directory: 'scons_local',
                     scm: [url: 'https://github.com/01org/isa-l.git',
                           branch: "${env.ISAL_COMMIT}",
                           cleanAfterCheckout: true],
                     no_install: true,  // No separate install step
                     TARGET_PREFIX: '/testbin',
                     target_work: 'testbin'
 
          echo "isal build succeeded"
        } // steps
      } // stage ('CentOS7 isal prebuild')
    } // parallel
    } // stage ('phase 1')
    stage('Phase 2') {
    parallel {
      stage ('CentOS7 cart build') {
        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {
          sconsBuild target: 'cart',
                     scm: [url: 'https://github.com/daos-stack/cart.git',
                           cleanAfterCheckout: true,
                           withSubmodules: true]
          echo "CaRT build succeeded"
        } // steps
      } // stage ('CentOS7 cart build')
/*
      stage ('CentOS7 iof build') {
        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {
          sconsBuild target: 'iof',
                     scm: [url: 'https://github.com/daos-stack/iof.git',
                           cleanAfterCheckout: true,
                           withSubmodules: true]
          echo "IOF build succeeded"
        } // steps

      } // stage ('CentOS7 IOF build')
*/      stage ('CentOS7 basic checks') {

        agent {
          dockerfile {
            filename 'Dockerfile.centos.7'
            dir 'scons_local/docker'
            label 'docker_runner'
            additionalBuildArgs '$BUILDARGS'
          }
        } // agent
        steps {
          //checkout scm
          // Older scons_local_review only used master branch.
          // Newer one looked up the last known good build of master branch.
          // Pipeline currently does not have access that info.
          checkoutScm url: 'http://git.mcs.anl.gov/radix/openpa.git',
                      branch: "${env.OPENPA_COMMIT}",
                      checkoutDir: 'openpa',
                      cleanAfterCheckout: true
          checkoutScm url: 'https://github.com/mercury-hpc/mercury.git',
                      branch: "${env.MERCURY_COMMIT}",
                      checkoutDir: 'mercury',
                      cleanAfterCheckout: true,
                      withSubmodules: true
          checkoutScm url: 'https://github.com/ofiwg/libfabric.git',
                      branch: "${env.OFI_COMMIT}",
                      checkoutDir: 'ofi',
                      cleanAfterCheckout: true
          checkoutScm url: 'https://github.com/pmix/master.git',
                      branch: "${env.PMIX_COMMIT}",
                      checkoutDir: 'pmix',
                      cleanAfterCheckout: true
          checkoutScm url: 'https://github.com/open-mpi/ompi.git',
                      branch: "${env.OMPI_COMMIT}",
                      checkoutDir: 'ompi',
                      cleanAfterCheckout: true
          // Need docker >= 17.12 to use dir { sh () }
          sh('''rm -rf /testbin/*
                rm -rf testbin
                for new_dir in openpa hwloc; do
                  mkdir -p testbin/${new_dir}
                  ln -s ${PWD}/testbin/${new_dir} /testbin/${new_dir}
                done''')
          unstash 'CentOS7-hwloc'
          unstash 'CentOS7-openpa'
          script {
            cmds2 = '''export WORKSPACE=""
                       export prebuilt1="PREBUILT_PREFIX=/testbin/hwloc"
                              prebuilt1+=":/testbin/openpa"
                       export prebuilt2="HWLOC_PREBUILT=/testbin/hwloc"
                              prebuilt2+=" OPENPA_PREBUILT=/testbin/openpa"
                       export SRC_PREFIX="${PWD}"
                              SRC_PREFIX+=":${PWD}/scons_local/test/prereq"
                       pushd scons_local
                         ./test_scons_local.sh
                       popd'''
            rc = sh(script: cmds2, returnStatus: true)

            // reference https://issues.jenkins-ci.org/browse/JENKINS-39203
            if (rc != 0) {
              echo "fail"
              stepResult(name: env.STAGE_NAME, context: "phase1",
                         result: "FAILURE")
            } else if (rc == 0) {
              echo "pass"
              stepResult(name: env.STAGE_NAME, context: "phase1",
                         result: "SUCCESS")
            }
          } // script

        } // steps
      } // stage ('CentOS7 basic checks')
      // Replicatate basic checks for Leap-15/Ubuntu-18
    } // parallel
    } // stage ('phase 2')
  } // stages
} // pipeline