pipeline {
    agent any
    options {
        timestamps()
    }
    stages {
        stage ('Build app WCG') {
            agent {
                dockerfile { 
                    filename 'Dockerfile.Golang' 
                    args '--network host'
               }
            }
            steps {
                git url: 'https://github.com/maxmax03/word-cloud-generator'
                sh '''
                make lint && make test
                 if [ $? -ne 0 ];
                    then
                        echo "Test error"
                        exit 1
                    fi
                export GOPATH=$WORKSPACE/go
                export PATH="$PATH:$(go env GOPATH)/bin"               
                go get github.com/tools/godep
                go get github.com/smartystreets/goconvey
                go get github.com/GeertJohan/go.rice/rice  
                go get github.com/wickett/word-cloud-generator/wordyapi
                go get github.com/gorilla/mux
                sed -i "s/1.DEVELOPMENT/1.$BUILD_NUMBER/g" static/version
                GOOS=linux GOARCH=amd64 go build -o ./artifacts/word-cloud-generator -v 
                md5sum artifacts/word-cloud-generator
                pwd
                ls -l artifacts/
                gzip -f ./artifacts/word-cloud-generator
                cat static/version
                '''
                nexusArtifactUploader ( 
                  artifacts: [[
                    artifactId: 'word-cloud-generator', 
                    classifier: '', file: 'artifacts/word-cloud-generator.gz', 
                    type: 'gz']], 
                  credentialsId: 'nexus-uploader', 
                  groupId: "$git_branch", 
                  nexusUrl: 'localhost:8081', 
                  nexusVersion: 'nexus3', 
                  protocol: 'http', 
                  repository: 'word-cloud-build', 
                  version: '1.$BUILD_NUMBER'
                )
            }
        }
        stage('Upload for tests') {
            agent {
                dockerfile { 
                    filename 'Dockerfile.Alpine' 
                    args '--network host'
               }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus_downloader', 
                        usernameVariable: 'nexus_user', 
                        passwordVariable: 'nexus_password'
                    )])
                {
                  sh '''
                   curl -u ${nexus_user}:${nexus_password} -X GET "http://localhost:8081/repository/word-cloud-build/$GIT_BRANCH/word-cloud-generator/1.$BUILD_NUMBER/word-cloud-generator-1.$BUILD_NUMBER.gz" -o /opt/wordcloud/word-cloud-generator.gz
                   if [[ $? -ne 0 ]];
                   then
                       echo "File not found"
                       exit 1
                   else
                       gunzip -f /opt/wordcloud/word-cloud-generator.gz
                       chmod +x /opt/wordcloud/word-cloud-generator
                       /opt/wordcloud/word-cloud-generator &
                       sleep 5
                       res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://localhost:8888/version | jq '. | length'`
                       if [[ "1" != "$res" ]]; then 
                          exit 98
                       fi
                       res=`curl -s -H "Content-Type: application/json" -d '{"text":"ths is a really really really important thing this is"}' http://localhost:8888/api | jq '. | length'`
                       if [[ "7" != "$res" ]]; then
                          exit 99
                       fi
                       sleep 90
                   fi
                  '''
                }
            }
        }
    }   
}
