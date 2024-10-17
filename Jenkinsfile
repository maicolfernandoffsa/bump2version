pipeline {
    agent {
        label 'docker'
    }

    environment {
        PRODUCT = 'ghcli'
        GIT_MAIN_BRANCH = 'main'
    }

    options {
       // ansiColor('xterm')
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        // Ⓐ Retrieve the project code from the repository. Extract the branch
        // name from the context of execution which can be a branch build or a pull
        // request build.
        stage('Checkout') {
            steps {
                script {
                    BRANCH_NAME = env.CHANGE_BRANCH ? env.CHANGE_BRANCH : env.BRANCH_NAME
                    deleteDir()
                    git url: "git@<githubHost>:<org>/<repo>.git", branch: BRANCH_NAME
                }
            }
        }

        // Ⓑ Prepare some variable and build the Docker Image
        stage('Prepare') {
            steps {
                script {
                    // Ⓑ① Generate a unique container name to avoid conflicts with multiple
                    // pull request jobs running in parallel. We also use the unique name for 
                    // image.
                    CONTAINER_NAME = env.PRODUCT + '_' + UUID.randomUUID().toString().replace("-", "")
                    IMAGE_BASE_TAG = env.PRODUCT + ':' + CONTAINER_NAME

                    // Ⓑ② Retrieve the previous version present in the file VERSION
                    LAST_VERSION = sh(
                        script: 'cat VERSION',
                        returnStdout: true
                    ).trim()

                    // Ⓑ③ To avoid creating releases each time something is pushed to the main branch,
                    // we prepare a variable to know if a release has to be created or not.
                    RELEASE_ON = sh(
                        script: 'if git rev-parse --verify -q HEAD^2 > /dev/null; then echo "y"; else echo "n"; fi',
                        returnStdout: true
                    ).trim().toBoolean() && BRANCH_NAME == env.GIT_MAIN_BRANCH

                    // Ⓑ④ Build the Docker Image
                    sh "docker build . -t ${IMAGE_BASE_TAG}"

                    // Ⓑ⑤ We create a non running container from the image to use it
                    // in next steps. This is a way to share the /usr/src across various
                    // Docker Containers.
                    sh "docker container create \
                        -v /usr/src \
                        --name ${CONTAINER_NAME} \
                        ${IMAGE_BASE_TAG}"
                }
            }
        }

        // Ⓒ We run the tests with the 
        stage('Test') {
            steps {
                script {
                // Ⓒ① We run the tests with the previous container as a volume
                // to make sure the test reports are collected by the Qualify step
                sh "docker run \
                    --tty \
                    --rm \
                    --volumes-from ${CONTAINER_NAME} \
                    ${IMAGE_BASE_TAG} \
                    /usr/bin/make test"
                }
            }
        }

        // Ⓓ Run the quality gate
        stage('Qualify') {
            steps {
                withCredentials([string(credentialsId: '<credsId>', variable: 'SONAR_LOGIN')]) {
                    script {
                        // Ⓓ① Make sure the sonar execution get the correct arguments 
                        // regarding the context of execution: main, pr, ...
                        if (env.CHANGE_ID != null) {
                            options = "-Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} "
                            options += " -Dsonar.pullrequest.key=${env.CHANGE_ID} "
                            options += " -Dsonar.pullrequest.base=${env.CHANGE_TARGET}"
                        } else if (BRANCH_NAME == 'main') {
                            options = ''
                        } else {
                            options = "-Dsonar.branch.name=${BRANCH_NAME}"
                        }

                        // Ⓓ② Execute the quality analysis with the volume that 
                        // has been used for the tests. It will let the analyser 
                        // to gather the test results.
                        sh "docker run \
                            --rm \
                            --volumes-from ${CONTAINER_NAME} \
                            -e SONAR_HOST_URL=https://<sonarHost> \
                            -e SONAR_LOGIN=${SONAR_LOGIN} \
                            sonarsource/sonar-scanner-cli \
                            sonar-scanner ${options}"
                    }
                }
            }
        }

        // Ⓔ In case we need to create a release, we first need to
        // set the new version.
        stage('Version') {
            // Ⓔ① Check if the job is a situation that require to build
            // and publish a new release
            when {
                expression { return RELEASE_ON }
            }

            steps {
                script {
                    // Ⓔ② Execute the make task "bump-minor" to automatically
                    // update the files: VERSION, scripts/ghcli, .bumpversion.cfg
                    // It will bump the minor. Ex: 0.1 -> 0.2
                    // We also give a name to the container to make it easier to 
                    // commit the result (create new image from a container)
                    sh "docker run \
                        --tty \
                        --name ${CONTAINER_NAME}-release \
                        -e CURRENT_VERSION=${LAST_VERSION} \
                        ${IMAGE_BASE_TAG} \
                        /usr/bin/make bump-minor"

                    // Ⓔ③ We create a new image from the docker container
                    DOCKER_IMAGE_COMMIT = sh(
                        script: "docker commit ${CONTAINER_NAME}-release",
                        returnStdout: true
                    ).trim()

                    // Ⓔ④ We retrieve the updated files from the container to the 
                    // current Jenkins workspace. We will commit them later.
                    sh "docker cp ${CONTAINER_NAME}:/usr/src/VERSION ."
                    sh "docker cp ${CONTAINER_NAME}:/usr/src/scripts/ghcli scripts/"
                    sh "docker cp ${CONTAINER_NAME}:/usr/src/.bumpversion.cfg ."

                    // Ⓔ⑤ We do not need the container used to bump the version
                    sh "docker rm ${CONTAINER_NAME}-release"
                }
            }
        }

        // Ⓕ With the new Docker Image, we can publish it to a 
        // Docker Registry to make it available for download.
        stage('Publish') {
            when {
                expression { return RELEASE_ON }
            }

            steps {
                script {
                    // Ⓕ① We retrieve the new version from the copied VERSION file
                    NEW_VERSION = sh (script: 'cat VERSION', returnStdout: true).trim()

                    // Ⓕ② We tag the image with this new version
                    sh "docker tag \
                        ${DOCKER_IMAGE_COMMIT} \
                        <internalRegistryHost>/component-releases/${env.PRODUCT}:${NEW_VERSION}"

                    // Ⓕ③ We tag the image to make it the latest version as well
                    sh "docker tag \
                        ${DOCKER_IMAGE_COMMIT} \
                        <internalRegistryHost>/component-releases/${env.PRODUCT}:latest"
                }

                withCredentials([usernamePassword(credentialsId: '<credsId>', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    // Ⓕ④ For our internal registry, we first need to login
                    sh "docker login \
                        --username $USERNAME \
                        --password $PASSWORD \
                        <internalRegistryHost>"

                    // Ⓕ⑤ Finally, we can push to the registry both tags
                    sh "docker push <internalRegistryHost>/component-releases/${env.PRODUCT}:${NEW_VERSION}"
                    sh "docker push <internalRegistryHost>/component-releases/${env.PRODUCT}:latest"
                }
            }
        }

        // Ⓖ Finally, we can manage the last part of Git management
        stage('Release') {
            when {
                expression { return RELEASE_ON }
            }

            steps {
                script {
                    // Ⓖ① We add the files that have been updated with
                    // the new version.
                    sh "git add VERSION .bumpversion.cfg scripts/ghcli"
                    sh "git commit -m 'Release ${NEW_VERSION}'"

                    // Ⓖ② We create the git tag
                    sh "git tag v${NEW_VERSION}"

                    // Ⓖ③ We now configure the upstream (related to Jenkins)
                    // and we push the commit and the tag to the remote origin
                    sh "git branch -u origin/${env.GIT_MAIN_BRANCH}"
                    sh "git push && git push --tags"
                }
            }
        }
    }

    // Ⓗ After all of that, we do some house cleaning by removing
    // Docker Image and Docker Container used for the build. The 
    // workspace is also cleared.
    post {
        always {
            script {
                sh "docker rm ${CONTAINER_NAME}"
                sh "docker rmi ${env.PRODUCT}:${CONTAINER_NAME}"
            }

            deleteDir()
        }
    }
}
