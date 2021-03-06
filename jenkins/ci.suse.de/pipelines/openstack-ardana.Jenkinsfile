/**
 * The openstack-ardana Jenkins Pipeline
 */

pipeline {
  options {
    // skip the default checkout, because we want to use a custom path
    skipDefaultCheckout()
    timestamps()
    // reserve a resource if instructed to do so, otherwise use an empty resource list
    lock(
      variable: 'reserved_env',
      extra: (reserve_env == 'true' ? [[label: cloud_env, quantity: 1]] : [])
    )
  }

  agent {
    node {
      label 'cloud-ci'
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          // Set this variable to be used by upstream builds
          env.blue_ocean_buildurl = env.RUN_DISPLAY_URL
          env.cloud_type = "virtual"
          if (cloud_env == '') {
            error("Empty 'cloud_env' parameter value.")
          }
          if (updates_test_enabled == 'true' && maint_updates != '') {
            error("Maintenance updates and test-updates cannot be applied at the same time.")
          }
          if (reserve_env == 'true') {
            echo "Reserved resource: " + env.reserved_env
            if (env.reserved_env && reserved_env != null) {
              env.cloud_env = reserved_env
            } else {
              error("Jenkins bug (JENKINS-52638): couldn't reserve a resource with label $cloud_env")
            }
          }
          currentBuild.displayName = "#${BUILD_NUMBER}: ${cloud_env}"
          if ( cloud_env.startsWith("qe") || cloud_env.startsWith("pcloud") || cloud_env.startsWith("hw-") ) {
              env.cloud_type = "physical"
          }
          // Parameters of the type 'extended-choice' are set to null when the job
          // is automatically triggered and its value is set to ''. So, we need to set
          // it to '' to be able to pass it as a parameter to downstream jobs.
          if (env.tempest_filter_list == null) {
            env.tempest_filter_list = ''
          }
          if (env.qa_test_list == null) {
            env.qa_test_list = ''
          }
          sh('''
            git clone $git_automation_repo --branch $git_automation_branch automation-git
            cd automation-git

            if [ -n "$github_pr" ] ; then
              scripts/jenkins/cloud/pr-update.sh
            fi
          ''')
          cloud_lib = load "$WORKSPACE/automation-git/jenkins/ci.suse.de/pipelines/openstack-cloud.groovy"
          cloud_lib.load_os_params_from_resource(cloud_env, reserve_env.toBoolean())
          cloud_lib.load_extra_params_as_vars(extra_params)
          cloud_lib.ansible_playbook('load-job-params',
                                      "-e jjb_type=job-template -e jjb_file=$WORKSPACE/automation-git/jenkins/ci.suse.de/templates/cloud-ardana-pipeline-template.yaml"
                                      )
          cloud_lib.ansible_playbook('notify-rc-pcloud')
        }
      }
    }

    stage('Prepare input model') {
      steps {
        script {
          if (scenario_name != '') {
            cloud_lib.ansible_playbook('generate-cloud')
          } else {
            cloud_lib.ansible_playbook('clone-input-model')
          }
        }
      }
    }

    stage('Generate heat template') {
      when {
        expression { cloud_type == 'virtual' }
      }
      steps {
        script {
          cloud_lib.ansible_playbook('generate-heat-template')
        }
      }
    }

    stage('Prepare infra and build package(s)') {
      // abort all stages if one of them fails
      failFast true
      parallel {

        stage('Start bare-metal deployer VM') {
          when {
            expression { cloud_type == 'physical' }
          }
          steps {
            script {
              cloud_lib.ansible_playbook('start-deployer-vm')
            }
          }
        }

        stage('Create heat stack') {
          when {
            expression { cloud_type == 'virtual' }
          }
          steps {
            script {

              // Needed to pass the generated heat template file contents as a text parameter value
              def heat_template = sh (
                returnStdout: true,
                script: 'cat "$WORKSPACE/heat-stack-${scenario_name}${model}.yml"'
              )

              cloud_lib.trigger_build("openstack-cloud-heat-$os_cloud", [
                string(name: 'cloud_env', value: "$cloud_env"),
                string(name: 'heat_action', value: "create"),
                text(name: 'heat_template', value: heat_template),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                string(name: 'os_project_name', value: "$os_project_name"),
                text(name: 'extra_params', value: extra_params)
              ], false)
            }
          }
        }

        stage('Build test packages') {
          when {
            expression { gerrit_change_ids != '' || update_gerrit_change_ids != '' || upgrade_gerrit_change_ids != '' }
          }
          steps {
            script {
              def slaveJob = cloud_lib.trigger_build('openstack-ardana-testbuild-gerrit', [
                string(name: 'gerrit_change_ids', value: "$gerrit_change_ids"),
                string(name: 'update_gerrit_change_ids', value: "$update_gerrit_change_ids"),
                string(name: 'upgrade_gerrit_change_ids', value: "$upgrade_gerrit_change_ids"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                text(name: 'extra_params', value: extra_params)
              ], false)

              if (gerrit_change_ids != '') {
                env.deploy_test_repository_url = "http://download.suse.de/ibs/Devel:/Cloud:/Testbuild:/ardana-ci-${slaveJob.getNumber()}-deploy/standard"
                if (extra_repos == '') {
                  env.extra_repos = deploy_test_repository_url
                } else {
                  env.extra_repos = "${deploy_test_repository_url},${extra_repos}"
                }
              }

              if (update_gerrit_change_ids != '') {
                env.update_test_repository_url = "http://download.suse.de/ibs/Devel:/Cloud:/Testbuild:/ardana-ci-${slaveJob.getNumber()}-update/standard"
                if (update_extra_repos == '') {
                  env.update_extra_repos = update_test_repository_url
                } else {
                  env.update_extra_repos = "${update_test_repository_url},${update_extra_repos}"
                }
              }

              if (upgrade_gerrit_change_ids != '') {
                env.upgrade_test_repository_url = "http://download.suse.de/ibs/Devel:/Cloud:/Testbuild:/ardana-ci-${slaveJob.getNumber()}-upgrade/standard"
                if (upgrade_extra_repos == '') {
                  env.upgrade_extra_repos = upgrade_test_repository_url
                } else {
                  env.upgrade_extra_repos = "${upgrade_test_repository_url},${upgrade_extra_repos}"
                }
              }
            }
          }
        }

      } // parallel
    } // stage('Prepare infra and build package(s)')

    stage('Setup SSH access') {
      steps {
        script {
          cloud_lib.ansible_playbook('setup-ssh-access')
          cloud_lib.get_deployer_ip()
        }
      }
    }

    stage('Prepare environment') {
      // abort all stages if one of them fails
      failFast true
      parallel {
        stage('Bootstrap CLM') {
          steps {
            script {
              cloud_lib.ansible_playbook('bootstrap-clm', "-e extra_repos='$extra_repos'")
            }
          }
        }
        stage('Deploy SES for vcloud') {
          when {
            expression { ses_enabled == 'true' && cloud_type == 'virtual' }
          }
          steps {
            script {
              retry(3) {
                cloud_lib.trigger_build('openstack-ses', [
                  string(name: 'ses_id', value: "$cloud_env"),
                  string(name: 'os_cloud', value: "$os_cloud"),
                  string(name: 'network', value: "${cloud_env}-cloud_management_net"),
                  string(name: 'git_automation_repo', value: "$git_automation_repo"),
                  string(name: 'git_automation_branch', value: "$git_automation_branch"),
                  string(name: 'os_project_name', value: "$os_project_name")
                ], false)
              }
            }
          }
        }
      }
    }

    stage('Bootstrap nodes') {
      steps{
        script {
          if (cloud_type == 'virtual') {
            if (pxe_boot_enabled == 'true') {
              cloud_lib.ansible_playbook('bootstrap-osbmc-nodes')
            } else {
              cloud_lib.ansible_playbook('bootstrap-vcloud-nodes')
            }
          } else {
            cloud_lib.ansible_playbook('bootstrap-pcloud-nodes')
          }
        }
      }
    }

    stage('Deploy cloud') {
      when {
        expression { deploy_cloud == 'true' }
      }
      steps {
        script {
          cloud_lib.ansible_playbook('deploy-cloud')
          // list-or-diff-installed-packages does not physical deployments
          if (cloud_type == 'virtual') {
            cloud_lib.ansible_playbook('list-or-diff-installed-packages', "-e wanted_action=list")
          }
        }
      }
    }

    stage('Update cloud') {
      when {
        expression { deploy_cloud == 'true' && update_after_deploy == 'true' && (maint_updates != '' || update_to_cloudsource != '') }
      }
      steps {
        script {
          // This is a mark for fail action 'collect list of installed packages'
          // to distinguish stages after "Update cloud" and do the diff of installed
          // packages. Can be removed when jenkins feature (JENKINS-48315) is merged.
          env.stage_after_update = "true"

          if (update_to_cloudsource == '') {
            cloud_lib.ansible_playbook('ardana-update')
          } else {
            cloud_lib.ansible_playbook('ardana-update', "-e cloudsource=$update_to_cloudsource -e extra_repos=$update_extra_repos")
          }

          // list-or-diff-installed-packages does not physical deployments
          if (cloud_type == 'virtual') {
            cloud_lib.ansible_playbook('list-or-diff-installed-packages', "-e wanted_action=diff")
          }
        }
      }
    }

    stage('Upgrade cloud') {
      when {
        expression { deploy_cloud == 'true' && update_after_deploy == 'true' && upgrade_cloudsource != '' }
      }
      steps {
        script {
          // This is a mark for fail action 'collect list of installed packages'
          // to distinguish stages after "Upgrade cloud" and do the diff of installed
          // packages. Can be removed when jenkins feature (JENKINS-48315) is merged.
          env.stage_after_update = "true"

          cloud_lib.ansible_playbook('ardana-disable-repos')
          cloud_lib.ansible_playbook('ardana-upgrade', "-e cloudsource=$upgrade_cloudsource -e extra_repos=$upgrade_extra_repos")

          // list-or-diff-installed-packages does not physical deployments
          if (cloud_type == 'virtual') {
            cloud_lib.ansible_playbook('list-or-diff-installed-packages', "-e wanted_action=diff")
          }
        }
      }
    }

    stage ('Prepare tests') {
      when {
        expression { tempest_filter_list != '' || qa_test_list != '' || want_caasp == 'true' || want_caaspv4 == 'true' }
      }
      steps {
        script {
          // Generate stages for Tempest tests
          cloud_lib.generate_tempest_stages(env.tempest_filter_list)
          // Generate stages for QA tests
          cloud_lib.generate_qa_tests_stages(env.qa_test_list)
          // Generate stage for CaaSP deployment
          if (want_caasp == 'true') {
            stage('Deploy CaaSP') {
              cloud_lib.ansible_playbook('deploy-caasp')
            }
          }
          // Generate stage for CaaSPv4 deployment
          if (want_caaspv4 == 'true') {
            stage('Deploy CaaSPv4') {
              cloud_lib.ansible_playbook('deploy-caasp-v4-terraform')
            }
          }
        }
      }
    }

  }

  post {
    always {
      script{
        sh('''
          source automation-git/scripts/jenkins/cloud/jenkins-helper.sh
          run_python_script automation-git/scripts/jenkins/jenkins-job-pipeline-report.py \
            --recursive \
            --filter 'Declarative: Post Actions' \
            --filter 'Setup workspace' > .artifacts/pipeline-report.txt || :
        ''')
        archiveArtifacts artifacts: ".artifacts/**/*", allowEmptyArchive: true
      }
      script{
        if (env.DEPLOYER_IP != null) {
          if (cloud_type == "virtual") {
            if (cleanup == "always" ||
                cleanup == "on success" && currentBuild.currentResult == "SUCCESS" ||
                cleanup == "on failure" && currentBuild.currentResult != "SUCCESS") {

              build job: "openstack-cloud-heat-$os_cloud", parameters: [
                string(name: 'cloud_env', value: "$cloud_env"),
                string(name: 'heat_action', value: "delete"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                string(name: 'os_project_name', value: "$os_project_name"),
                text(name: 'extra_params', value: extra_params)
              ], propagate: false, wait: false
            } else {
              if (os_project_name == 'cloud-ci') {
                echo """
******************************************************************************
** The deployer for the '${cloud_env}' virtual environment is reachable at:
**
**        ssh root@${DEPLOYER_IP}
**
** IMPORTANT: the '${cloud_env}' virtual environment may be (may have
** already been) deleted by any of the future periodic job runs. To prevent
** that, you should use the Lockable Resource page at
** https://ci.nue.suse.com/lockable-resources and reserve the '${cloud_env}'
** resource.
**
** Please remember to release the '${cloud_env}' Lockable Resource when
** you're done with the environment.
**
** You don't have to manually delete the heat stack if you don't want to.
**
******************************************************************************
                """
              } else {
                def cloud_url_text = "the cloud"
                if (os_cloud == 'engcloud') {
                  cloud_url_text="the engineering cloud at https://engcloud.prv.suse.net/project/stacks/"
                } else if (os_cloud == 'susecloud') {
                  cloud_url_text="the SUSE cloud at https://cloud.suse.de/project/stacks/"
                }
                echo """
******************************************************************************
** The deployer for the '${cloud_env}' virtual environment is reachable at:
**
**        ssh root@${DEPLOYER_IP}
**
** Please delete the '${cloud_env}-cloud' stack when you're done,
** by using one of the following methods:
**
**  1. log into ${cloud_url_text}
**  and delete the stack manually, or
**
**  2. (preferred) trigger a manual build for the openstack-cloud-heat-${os_cloud}
**  job at https://ci.nue.suse.com/job/openstack-cloud-heat-${os_cloud}/build and
**  use the same '${cloud_env}' cloud_env value and the 'delete' action for the
**  parameters
**
******************************************************************************
                """
              }
            }
          } else {
            echo """
******************************************************************************
** The deployer for the '${cloud_env}' physical environment is reachable at:
**
**        ssh root@${DEPLOYER_IP}
**
******************************************************************************
            """
          }
        }
      }
    }
    success {
      sh '''
        if [ -n "$github_pr" ] ; then
          automation-git/scripts/cloud/pr-success.sh
        fi
      '''
    }
    failure {
      sh '''
        if [ -n "$github_pr" ] ; then
          automation-git/scripts/cloud/pr-failure.sh
        fi
      '''
      script {
        cloud_lib.track_failure()
        if (env.stage_after_update != null) {
          // list-or-diff-installed-packages does not physical deployments
          if (cloud_type == 'virtual') {
            cloud_lib.ansible_playbook('list-or-diff-installed-packages', "-e wanted_action=list -e state1=after_update")
          }
        } else {
          // list-or-diff-installed-packages does not physical deployments
          if (cloud_type == 'virtual') {
            cloud_lib.ansible_playbook('list-or-diff-installed-packages', "-e wanted_action=list")
          }
        }
        if (collect_supportconfig == 'true') {
          cloud_lib.ansible_playbook('collect-supportconfig')
        }
        archiveArtifacts artifacts: ".artifacts/**/*", allowEmptyArchive: true
      }
    }
    cleanup {
      cleanWs()
    }
  }
}
