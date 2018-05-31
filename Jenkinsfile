Date date = new Date()
env.MP_PART_NUMBER = date.format("yyMMdd") + "00" + "NIG"
env.ENABLE_AUTO_FLASHING = 1
env.USED_DISPLAY_DEVICE = "Volvo_CSD"
env.BUILD_THREADS = 10
def BASE_URL = 'ssh://10.236.95.27:29418'
def GIT_BRANCH = 'android-p'

// These has to be moved to jenkins library
def getlatestCommitFromRepo(GIT_REPO_URL, GIT_BRANCH) {
    echo "the component now is ${GIT_REPO_URL}"
    //1.first step look on the component repo for the latest hash commit.
    git url: "$GIT_REPO_URL", branch: "$GIT_BRANCH"
    //2.Get the hash commit
    def latest_commit = sh(returnStdout: true, script: " git rev-parse HEAD").trim()
    return latest_commit
    }

def updateManifest(SELECTED_APPLICATION, GIT_REPO_URL, GIT_BRANCH, FILE_NAME, latest_commit, repo ) {
    def reviewers = "r=prasad.yedla@aptiv.com,"
         reviewers += "r=saulius.eidukas@aptiv.com,"
    def should_I_edit = true
    //Creating strings
    def base_prefix = 'path='
    def selected_component= "vendor/aptiv/components/${SELECTED_APPLICATION}"
    def prefix = base_prefix + '"'+selected_component+'"'
    echo prefix
    git url: "$GIT_REPO_URL", branch: "$GIT_BRANCH"
    //git url: 'ssh://10.236.95.27:29418/Android_bsd_manifest',  branch: "devel"
    if (fileExists("$FILE_NAME")) {
        echo "File found"
        echo "Check if manifest has already the latest hash"
        def searchText = "${prefix} revision=(.+) delphi-targets"
        def matcher = readFile("$FILE_NAME") =~  '"'+latest_commit+'"'
        if (matcher){
            should_I_edit = false
            echo 'Manifest contains the latest commit'
        }else {
                echo "Updating manifest with latest commit"
        }
    } else {
            echo 'File not found'
    }

    echo "latest hash $latest_commit"
    if(should_I_edit) {
        def latest_commit_with_double_quotes = '"'+latest_commit+'"'
        def replaceText = "${prefix} revision=${latest_commit_with_double_quotes}  delphi-targets"
        def readContent =  readFile("$FILE_NAME")
        def fileText = (readContent =~ prefix + ' revision=(.+) delphi-targets').replaceFirst( replaceText)
        writeFile file: "${FILE_NAME}", text: fileText
        try{
            sh "ls"
            sh "pwd"
            sh "git add $FILE_NAME"
            sh 'git status'
            sh 'gitdir=$(git rev-parse --git-dir); scp -p -P 29418 jenkins_niu@10.236.95.27:hooks/commit-msg ${gitdir}/hooks/'
            sh "git commit -m \"${SELECTED_APPLICATION} binary repo bump in Pre-release Manifest based on Jenkins baseline_master Build - ${BUILD_NUMBER}\""
            sh "git push origin HEAD:${GIT_BRANCH}"
            echo 'Commit has been made :-)'
        } catch (Exception err) {
            echo "I can not make the commit ${err}"
        }
    }else {
            echo 'No commit'
    }
}

node('aosp') {
    try {
        def server = Artifactory.server 'GOT artifactory'
        def buildInfo = Artifactory.newBuildInfo()
        buildInfo.env.capture = true
        server.bypassProxy = true
        def uploadSpec = """{
          "files": [
           {
            "pattern": "upload_items/*",
            "target": "ihu-android-p/nightly/${BUILD_NUMBER}/"
           }
         ]
       }"""
       stage('Fetch source code'){
          timestamps {
            echo '#### Fetching source code ####'
            dir('high'){
                sh 'repo init -u ssh://10.236.95.27:29418/Android_bsd_manifest.git -b devel  --reference=/opt/exws/aosp/android-p-ihu -m IHU_android-P-devel.xml --repo-url=ssh://10.236.95.27:29418/git-repo --no-repo-verify'
                sh 'repo sync -c -d -q'
            }
            sh 'rm -rf upload_items && mkdir upload_items'
            echo '###########################'
            echo '#### Fetching aosp tools repo ####'
            dir('high/vendor/aptiv/tools/android_aosp_tools') {
                git branch: 'android-p', url: 'ssh://10.236.95.27:29418/android_aosp_tools'
            }
            echo '###########################'
            echo '#### Copy builspec.mk to root directory ####'
            sh 'cp high/vendor/aptiv/tools/android_aosp_tools/build/buildspec.mk high/'
            echo '###########################'
          }apm install language-jenkinsfile
        }
        stage('Extracting Copyrightheaders'){
            dir('high'){
                sh '''#!/bin/bash
                export PATH="$HOME/bin:$PATH"
                PATHS_MANIFEST="$(paths_from_manifest.sh .repo/manifest.xml)"
                copyrightheaders.sh --input ./ --output ${WORKSPACE}/upload_items/ --silent ${PATHS_MANIFEST}'''
            }
        }
        stage('Apply Patched'){
            echo '#### Applying patches ####'
            sh './high/delphi_volvoihu_aosp_patches/apply-patch.sh '
            echo '###########################'
        }
        stage('Build Image'){
            withEnv(['ANDROID_TARGET_NAME=ihu_kraken', 'ANDROID_TARGET_TYPE=userdebug']) {
                timestamps {
                    try{
                        echo '#### Building High variant ####'
                        sh "#!/bin/bash \n" +
                        "cd high \n" +
                        "source build/envsetup.sh \n" +
                        "lunch ${ANDROID_TARGET_NAME}-${ANDROID_TARGET_TYPE} \n" +
                        "make droid dist -j${BUILD_THREADS} |tee ${ANDROID_TARGET_NAME}_${BUILD_NUMBER}.log"
                        sh "cp high/out/target/product/ihu_abl_car/${ANDROID_TARGET_NAME}-flashfiles-${BUILD_NUMBER}.zip upload_items/"
                        sh "cp high/out/target/product/ihu_abl_car/${ANDROID_TARGET_NAME}-ota-${BUILD_NUMBER}.zip upload_items/"
                        sh "cp high/out/dist/mp-android-${MP_PART_NUMBER}.VBF upload_items/"
                        warnings canComputeNew: false, canResolveRelativePaths: false, categoriesPattern: '', consoleParsers: [[parserName: 'Clang (LLVM based)']],
                        defaultEncoding: '', excludePattern: '*[MID]*', healthy: '', includePattern: '**/aptiv/components/**', messagesPattern: '', unHealthy: ''
                    }
                    catch (Exception e){
                        echo "### HIGH build failed. Please check the log for additional information ###"
                        currentBuild.result = 'FAILURE'
                        error 'High build failed'
                    }
                    finally{
                        archive "high/${ANDROID_TARGET_NAME}_${BUILD_NUMBER}.log"
                    }
                }
            }
        }
        stage('Extract Tuner'){
            // Creating Documentation
            sh "cd high/vendor/aptiv/components/vcc_ihu_tuner/Android/interface_java \n"+
            "rm -rf ./html \n" +
            "doxygen \n" +
            "tar -cvzf com.aptiv.tuner.doc.${BUILD_NUMBER}.tar.gz ./html \n" +
            "mv -vf com.aptiv.tuner.doc.${BUILD_NUMBER}.tar.gz ${WORKSPACE}/upload_items/com.aptiv.tuner.doc.${BUILD_NUMBER}.tar.gz"
            // Extracting jar
            sh "cp high/out/target/product/ihu_abl_car/com.aptiv.tuner.jar/com.aptiv.tuner.jar ${WORKSPACE}/upload_items/"
        }
        stage('Build test packages'){
            withEnv(['ANDROID_TARGET_NAME=ihu_kraken', 'ANDROID_TARGET_TYPE=userdebug', 'USED_DISPLAY_DEVICE=Volvo_CSD']) {
                timestamps {
                    try{
                        echo '#### Creating test packages ####'
                        sh "#!/bin/bash \n" +
                            "cd high \n" +
                            "source build/envsetup.sh \n" +
                            "lunch ${ANDROID_TARGET_NAME}-${ANDROID_TARGET_TYPE} \n" +
                            "make delphi-tradefed-all -j${BUILD_THREADS} | tee delphi_tradefed_${ANDROID_TARGET_NAME}_${BUILD_NUMBER}.log \n" +
                            "make delphi-tradefed-dist"
                        sh "cp high/out/dist/delphi-tradefed-tests.zip upload_items/delphi-tradefed-tests-${BUILD_NUMBER}.zip"
                    }
                    catch (Exception e){
                        echo "### Test package build failed. Please check the log for additional information ###"
                        currentBuild.result = 'FAILURE'
                        error 'Failed  to build test package'
                    }
                    finally{
                        archive "high/delphi_tradefed_${ANDROID_TARGET_NAME}_${BUILD_NUMBER}.log"
                    }
                }
            }
        }
        stage('Creating baseline text'){
           checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'android_ci']], submoduleCfg: [], userRemoteConfigs: [[url: 'ssh://10.236.95.27:29418/android_ci']]])
            sh 'echo "VCC_IHU40_ANDROID_$(date +%y%W)_MP_${RELEASE_VERSION}_${BUILD_NUMBER}_baseline_VIP_${VIP_VERSION}"|tr -d "[:space:]" > version_name.properties'
            env.VERSION_NAME = readFile 'version_name.properties'
            sh "python3 ./android_ci/Jenkins-scripts/baseline_gen.py -f high/.repo/manifests/IHU_android-P-devel.xml -v ${VERSION_NAME}"
            sh "cp baseline_${VERSION_NAME}.json upload_items/baseline_VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}.json"
            try {
                sh 'wget -O old_base.json http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/$( expr $BUILD_NUMBER - "1" )/baseline_VCC-IHU40-COMB-NB-P-$( expr $BUILD_NUMBER - "1" ).json'
                sh 'python3 android_ci/Jenkins-scripts/changelog_gen.py -f old_base.json baseline_${VERSION_NAME}.json'
                sh 'cp changelog_${VERSION_NAME}.txt upload_items/changelog_VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}.txt'
            }
            catch(Exception err){
                echo "Changelog failed"
            }

        }
        stage('Updating Jira with fixed bugs'){
            try {
                sh 'python3 android_ci/Jenkins-scripts/changelogBilly.py SD_GOT_AD g5pgfNyY upload_items/changelog_VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}.txt'
            }
            catch(Exception err){
                echo "Failed to update Jira with bugs"
            }
        }
        stage('Extracting android.car.jar'){
            echo '#### Extracting android.car.jar ####'
            sh 'cp high/out/target/common/obj/JAVA_LIBRARIES/android.car_intermediates/classes.jar upload_items/android.car.jar'
            echo '###########################'
        }
        stage('Binary Package Generation'){
            echo '#### Generating Tuner Manager binary Packages ####'
            echo '#### Fetching binaries_android_tunermanager repo ####'
            dir('high/vendor/aptiv/components/_converted/vcc_ihu_tuner') {
                git branch: 'android-p', url: 'ssh://10.236.95.27:29418/binaries_android_tunermanager'
            }
            echo '###########################'
            sh './high/vendor/aptiv/components/vcc_ihu_tuner/export_android_binaries.sh high/out/target/product/ihu_abl_car/ high/vendor/aptiv/components/_converted/vcc_ihu_tuner/'
            dir('high/vendor/aptiv/components/_converted/vcc_ihu_tuner') {
                try{
                    sh 'git add --all'
                    sh 'git commit -m "Updated Tuner Binary modules - Jenkins baseline_master Build - ${BUILD_NUMBER}" -m "[No-Jira]"'
                    sh 'git push origin HEAD:android-p'
                }
                catch (Exception e){
                        echo "### Nothing to Commit ###"
                }
            }
            echo '###########################'

            echo '#### Generating AP_List binary Packages ####'
            echo '#### Fetching binaries_AP_List repo ####'
            dir('high/vendor/aptiv/components/_converted/liblist') {
                git branch: 'android-p', url: 'ssh://10.236.95.27:29418/binaries_AP_List'
            }
            echo '###########################'
            sh './high/vendor/aptiv/components/liblist/export_android_binaries.sh high/out/target/product/ihu_abl_car/ high/vendor/aptiv/components/_converted/liblist'
            dir('high/vendor/aptiv/components/_converted/liblist') {
                try{
                    sh 'git add --all'
                    sh 'git commit -m "Updated AP_List modules - Jenkins baseline_master Build - ${BUILD_NUMBER}" -m "[No-Jira]"'
                    sh 'git push origin HEAD:android-p'
                }
                catch (Exception e){
                        echo "### Nothing to Commit ###"
                }
            }

            echo '###########################'
            echo '#### Generating android_traffic binary Packages ####'
            echo '#### Fetching binaries_android_traffic repo ####'
            dir('high/vendor/aptiv/components/_converted/traffic') {
                git branch: 'android-p', url: 'ssh://10.236.95.27:29418/binaries_android_traffic'
            }
            echo '############################################################################################################'
            sh './high/vendor/aptiv/components/traffic/export_android_binaries.sh high/out/target/product/ihu_abl_car/ high/vendor/aptiv/components/_converted/traffic/'
            dir('high/vendor/aptiv/components/_converted/traffic') {
                try{
                    sh 'git add --all'
                    sh 'git commit -m "Updated Traffic Binary modules - Jenkins baseline_master Build - ${BUILD_NUMBER}" -m "[No-Jira]"'
                    sh 'git push origin HEAD:android-p'
                }
                catch (Exception e){
                        echo "### Nothing to Commit ###"
                }
            }
            echo '############################################################################################################'

            echo '#### Generating Binary Packages ####'
            sh 'find high/out/target/product/ihu_abl_car/proprietary_packages/ -type f -name "*.sh" -exec bash "{}" ";"'
            echo '###########################'
        }
        stage('Manifest Bump') {
            dir('vendor/aptiv/components/_converted/manifestbump') {
                def app_list = ['most', 'audioengine', 'audio', 'vcc_ihu_tuner', 'traffic', 'liblist']
                def binaries_repo_list = ['binaries_android_most', 'binaries_android_audioengine', 'binaries_android_audio', 'binaries_android_tunermanager', 'binaries_android_traffic', 'binaries_AP_List']
                if( currentBuild.result == null){
                    for(int i = 0; i < app_list.size(); i++) {
                        def SELECTED_APPLICATION = app_list[i]
                        echo "selected application is ${SELECTED_APPLICATION}"
                        def binaries_repo = binaries_repo_list[i]
                        def repo = "$BASE_URL/$binaries_repo"
                        echo "repo is ${repo}"
                        def latest_commit = getlatestCommitFromRepo("${repo}","$GIT_BRANCH")
                        echo "this is the latest hash: ${latest_commit}"
                        if (latest_commit != 0 && repo != null) {
                            echo "Bumping Manifest for Android-P - ${binaries_repo} repo"
                            updateManifest(SELECTED_APPLICATION, "$BASE_URL/Android_bsd_manifest", 'devel', 'IHU_android-P-prerelease.xml', latest_commit, repo )
                        }
                        else
                            echo "Manifest not bumped."
                    }
                }
            }
        }
        stage('Customer Build'){
            withEnv(['ANDROID_TARGET_NAME=ihu_kraken', 'ANDROID_TARGET_TYPE=userdebug']) {
                timestamps {
                    try{
                        echo '#### Fetching source code ####'
                        dir('high'){
                                sh "rm -rf vendor/aptiv/components/_converted"
                                sh 'repo init -u ssh://10.236.95.27:29418/Android_bsd_manifest.git -b devel  --reference=/opt/exws/aosp/android-p-ihu -m IHU_android-P-prerelease.xml --repo-url=ssh://10.236.95.27:29418/git-repo --no-repo-verify'
                                sh 'repo sync --force-sync'
                        }
                        echo '###########################'

                        echo '#### Applying patches ####'
                        sh './high/delphi_volvoihu_aosp_patches/apply-patch.sh '
                        echo '###########################'

                        echo '#### Building High - Customer variant ####'
                        sh "#!/bin/bash \n" +
                        "cd high \n" +
                        "source build/envsetup.sh \n" +
                        "lunch ${ANDROID_TARGET_NAME}-${ANDROID_TARGET_TYPE} \n" +
                        "make clean \n" +
                        "make droid dist -j${BUILD_THREADS} |tee ${ANDROID_TARGET_NAME}_CustomerBuild_${BUILD_NUMBER}.log"
                        sh "cp high/out/target/product/ihu_abl_car/${ANDROID_TARGET_NAME}-flashfiles-${BUILD_NUMBER}.zip upload_items/${ANDROID_TARGET_NAME}-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip"
                    }
                    catch (Exception e){
                        echo "### HIGH Customer build failed. Please check the log for additional information ###"
                        currentBuild.result = 'FAILURE'
                        error 'High build failed'
                    }
                    finally{
                        archive "high/${ANDROID_TARGET_NAME}_CustomerBuild_${BUILD_NUMBER}.log"
                    }
                }
            }
        }
        stage('Tag Manifest'){
            currentBuild.description = "version: $VERSION_NAME <br /> tag: VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}"
                try{
                    dir('high/.repo/manifests/'){
                        sh "git tag VCC-IHU40-COMB-NB-P-${BUILD_NUMBER} HEAD && git push origin refs/tags/VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}"
                    }
                }
                catch(Exception err){
                    echo "Tagging failed"
                }
        }
        stage('Generate Unified Manifest'){
            echo '#### Creating Combined Source code Manifest file ####'
            dir('high'){
               sh "repo manifest -o - -r --suppress-upstream-revision > ${WORKSPACE}/high/Customer_Build.xml"
             }            

            echo '#### Generate Aptiv Unified Manifest using XmlParser tool ####'
            sh "python3 ./android_ci/Jenkins-scripts/vccXmlParser.py high/.repo/manifests/manifest-delphi-P.xml ${WORKSPACE}/high/Customer_Build.xml"
            sh "mv output-unified.xml high/.repo/manifests/manifest-delphi-P.xml"

            echo '################### Push the Generated Unified Manifest to git repo ############################################'git credentialsId: 'd9965bc3-c74f-49e5-b051-5418d8835cda', url: 'https://github.com/markhulia/CI.git'
            dir('high/.repo/manifests/') {
                try{
                    sh 'git add --all'
                    sh 'git commit -m "Updated Unified Manifest for VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}" -m "[No-Jira]"'
                    sh 'git push origin HEAD:devel'
                }
                catch (Exception e){
                        echo "### Nothing to Commit ###"
                }
            }
        }
        stage('Uploading to the Artifactory/RCM'){
            parallel Artifactory: {
                try{
                    echo '### Uploading to the Artifactory ###'
                    server.upload(uploadSpec, buildInfo)
                    server.publishBuildInfo(buildInfo)
                    echo '###########################'
                }catch (Exception e){
                    echo 'Artifactory upload failed'
                    error 'Artifactory upload failed. Exiting.'
                }

            }, RCM: {
                try{
                    echo '### Uploading to the RCM ###'
                    withEnv(['TYPE_BUILD=HIGH','GOLDENCOMPILER=AllOfficial', 'PROJECT=VOLVO_IHU', 'SUBPROJECT=MP']){
                        sh './android_ci/RCM/rcm_transfer.py -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -i --vbra "IHU-$TYPE_BUILD" --vdes "<h2> <b>Android Master Baseline Build (Source & Customer)</b> </h2> <hr>  <a href=$BUILD_URL>Jenkins job</a>" --vstate "ongoing" --bstate "succeeded" --nbid '
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/ihu_kraken-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip" "IHU Android Customer Build Image" "application/x-gzip" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/ihu_kraken-flashfiles-${BUILD_NUMBER}.zip" "IHU Android HIGH Image" "application/x-gzip" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/ihu_kraken-ota-${BUILD_NUMBER}.zip" "IHU Android HIGH ota files" "application/x-gzip" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/android.car.jar" "IHU Android car jar" "application/octet-stream" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/baseline_VCC-IHU40-COMB-NB-P-${BUILD_NUMBER}.json" "Baseline text" "text/plain" "Text Info File"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/com.aptiv.tuner.doc.${BUILD_NUMBER}.tar.gz" "Tuner Documentation" "application/x-gtar" "Text Info File"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/com.aptiv.tuner.jar" "Tuner jar" "application/octet-stream" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -a -x "upload_items/mp-android-${MP_PART_NUMBER}.VBF" "MP VBF" "application/octet-stream" "Image"'
                        sh './android_ci/RCM/rcm_transfer.py -v 3 -p "$PROJECT" -s "$SUBPROJECT" -l "$VERSION_NAME" -i --vstate "succeeded" --bstate "succeeded"'
                    echo '###########################'}
                }catch (Exception e){
                        echo "RCM UPLOAD FAILED"
                    }
            }, TCI:{
                try {
                    echo '### Uploading to the TCI ###'
                    build job: 'Freestyle_jobs/upload_TCI_vcc-ihu_baseline_master', parameters: [string(name: 'BASELINE_NR', value: "${BUILD_NUMBER}")]
                }
                catch (Exception e){
                    echo "Upload to TCI failed"
                }
            },failFast: false

        }
        parallel ciTest: {
            stage('Trigger CI Auto Test') {
                try {
                    build job: 'Android_tests/vcc-ihu_android_ci-auto-test', parameters:
                        [string(name: 'ImageArtifactUrl', value: "http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/${BUILD_NUMBER}/ihu_kraken-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip"),
                         string(name: 'TestArtifactUrl', value: "http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/${BUILD_NUMBER}/delphi-tradefed-tests-${BUILD_NUMBER}.zip"),
                         string(name: 'TestNode', value: "at-at_ciat_ihu && android-p && at-at_audio")], wait: false
                }
                catch (Exception e) {
                    echo "Triggering sanity check failed"
                }
            }
            stage('Trigger CTS') {
                try {
                    build job: 'Android_tests/vcc-ihu_android_CTS_nightly', parameters:
                        [string(name: 'ImageArtifactUrl', value: "http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/${BUILD_NUMBER}/ihu_kraken-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip")], wait: false
                }
                catch (Exception e) {
                   echo "Triggering CTS failed"
                }
            }
            stage('Trigger VTS') {
                try {
                    build job: 'Android_tests/vcc-ihu_android_VTS_nightly', parameters:
                        [string(name: 'ImageArtifactUrl', value: "http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/${BUILD_NUMBER}/ihu_kraken-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip")], wait: false
                }
                catch (Exception e) {
                    echo "Triggering VTS failed"
                }
            }
            stage('Trigger BootUpTime Check') {
                try {
                    build job: 'Android_tests/vcc-ihu_android_BootTime_Check', parameters:
                        [string(name: 'ImageArtifactUrl', value: "http://10.236.88.232:8088/artifactory/ihu-android-p/nightly/${BUILD_NUMBER}/ihu_kraken-flashfiles-CustomerBuild-${BUILD_NUMBER}.zip")], wait: false
                }
                catch (Exception e) {
                    echo "Triggering BootUpTime Check failed"
                }
            }
        }, swdlTest: {
            stage('Trigger SWDL automated tests'){
                try {
                    sh 'curl "http://10.224.182.209/job/IHU_build_proxy_Android_P/buildWithParameters?token=START_IHU_SWDL_TEST&REMOTE_BUILD_NUMBER=${BUILD_NUMBER}"'
                }
                catch (Exception e) {
                    echo "Triggering SWDL automated tests failed!"
                }
            }
        }, failFast: false
   } // try ends
   catch (Exception e) {
          currentBuild.result = 'FAILURE'
          def reason = e.toString()
          env.FAILURECAUSE = e.toString()
          echo "Failure reason: ${FAILURECAUSE}"
          if (env.FAILURECAUSE == 'java.lang.InterruptedException' || env.FAILURECAUSE == 'hudson.AbortException: script returned exit code 143' ) {
              echo 'Job was aborted by user or new patchset'
          }
          if (env.FAILURECAUSE == 'java.io.IOException: Could not checkout') {
              echo 'Failed to checkout source code. Notifying Build Manager.'
              emailext attachLog: false,
              body: '''          Hello,
              $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:
              Reason for the build failure: Failed to check out source code, please check what is the issue ASAP.

              Console output at $BUILD_URL to view the results or attached build log.

              Have a nice day :)

              Best Regards,
              Jenkins''',
           subject: 'Jenkins cannot checkout sourcecode', to: "${CI_TEAM}"
          }
          if (env.FAILURECAUSE != 'java.lang.InterruptedException' &&
          env.FAILURECAUSE != 'java.io.IOException: Could not checkout' &&
          env.FAILURECAUSE != 'hudson.AbortException: script returned exit code 143') {
               emailext attachLog: false,
               body: '''          Hello,
                  $PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS:

                  Information about resolving failing Sanity Test please visit: http://confluenceprod1.delphiauto.net:8090/x/kg9f

                  Check console output at $BUILD_URL to view the results or attached build log.

                  Have a nice day :)

                  Best Regards,
                  Jenkins''',
               subject: 'Failed VCC IHU Android O Incremetal build', to: "${GERRIT_PATCHSET_UPLOADER_EMAIL}"
          }

      }
      finally {
        stage('Clean-up'){
             timestamps{
                cleanWs()
            }
        }
     }//end of node
}

