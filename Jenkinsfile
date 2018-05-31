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
        } catch (Exception err) {f node
}

            echo "I can not make the commit ${err}"
        }
    }else {
            echo 'No commit'
    }
}
