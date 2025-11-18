properties([
    pipelineTriggers([
        GenericTrigger(
            genericVariables: [
                [key: 'commits_modified', value: '$.commits[*].modified[*]'],
                [key: 'commits_added', value: '$.commits[*].added[*]'],
                [key: 'ref', value: '$.ref']
            ],
            causeString: 'Triggered by Yocto file changes',
            token: 'yocto-build-sync',
            regexpFilterText: '$commits_modified $commits_added',
            regexpFilterExpression: '.*(\\.bb|\\.bbappend|\\.conf|\\.inc|\\.bbclass).*',
            printContributedVariables: true,
            printPostContent: true
        )
    ]),
    parameters([
        string(name: 'VECTORSTORE', defaultValue: '/home/azureuser/rag-system/modular_code_base/vectorstore'),
        string(name: 'MARKER_FILE', defaultValue: '.yocto-full-sync-complete')
    ])
])

pipeline {
    agent any
    
    environment {
        VECTORSTORE = "${params.VECTORSTORE}"
        MARKER = "${params.VECTORSTORE}/${params.MARKER_FILE}"
        PATTERN = '\\.(bb|bbappend|conf|inc|bbclass)$'
    }
    
    options {
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timestamps()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Detect Mode') {
            steps {
                script {
                    def markerExists = sh(script: "test -f '${MARKER}' && echo 'yes' || echo 'no'", returnStdout: true).trim()
                    env.MODE = markerExists == 'yes' ? 'incremental' : 'full'
                    echo "→ Mode: ${env.MODE}"
                }
            }
        }
        
        stage('Find Files') {
            steps {
                script {
                    if (env.MODE == 'full') {
                        env.FILES = sh(script: "git ls-files | grep -E '${PATTERN}' || true", returnStdout: true).trim()
                    } else {
                        def prev = env.GIT_PREVIOUS_COMMIT ?: 'HEAD~1'
                        def changed = sh(script: "git diff --name-only ${prev} ${env.GIT_COMMIT} | grep -E '${PATTERN}' || true", returnStdout: true).trim()
                        env.FILES = changed ?: sh(script: "git ls-files | grep -E '${PATTERN}' || true", returnStdout: true).trim()
                    }
                    
                    def count = env.FILES ? env.FILES.split('\n').size() : 0
                    echo "→ Files: ${count}"
                    
                    if (count == 0) {
                        echo "ℹ No Yocto files to sync"
                        currentBuild.result = 'SUCCESS'
                        return
                    }
                }
            }
        }
        
        stage('Sync') {
            when { expression { env.FILES?.trim() } }
            steps {
                script {
                    def synced = 0
                    def failed = 0
                    
                    env.FILES.split('\n').each { file ->
                        if (file?.trim() && fileExists(file)) {
                            def destDir = "${VECTORSTORE}/yocto-files/${file.contains('/') ? file.substring(0, file.lastIndexOf('/')) : '.'}"
                            try {
                                sh "mkdir -p '${destDir}'"
                                sh "cp '${file}' '${destDir}/'"
                                synced++
                            } catch (e) {
                                echo "✗ ${file}"
                                failed++
                            }
                        }
                    }
                    
                    echo "→ Synced: ${synced}, Failed: ${failed}"
                    if (synced == 0 && failed > 0) error("All syncs failed")
                }
            }
        }
        
        stage('Mark Complete') {
            when { expression { env.MODE == 'full' && env.FILES?.trim() } }
            steps {
                sh "echo 'Full sync: \$(date)' > '${MARKER}'"
            }
        }
    }
    
    post {
        success { echo " ${env.MODE} sync: ${env.FILES?.split('\n')?.size() ?: 0} files" }
        failure { echo " Sync failed" }
    }
}
