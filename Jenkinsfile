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
            regexpFilterExpression: '.*(\\.bb|\\.bbappend|\\.conf|layer\\.conf|\\.inc|\\.bbclass).*',
            printContributedVariables: true,
            printPostContent: true
        )
    ]),
    parameters([
        string(name: 'RAG_BASE', defaultValue: '/home/azureuser/rag-system'),
        string(name: 'VECTORSTORE_DIR', defaultValue: 'modular_code_base/vectorstore'),
        string(name: 'PYTHON_ENV', defaultValue: '/home/azureuser/rag-system/venv')
    ])
])

pipeline {
    agent any
    
    environment {
        RAG_BASE = "${params.RAG_BASE}"
        VECTORSTORE = "${params.RAG_BASE}/${params.VECTORSTORE_DIR}"
        TEMP_DOCS = "${VECTORSTORE}/yocto-staging"
        PYTHON_ENV = "${params.PYTHON_ENV}"
        PATTERN = '\\.(bb|bbappend|conf|inc|bbclass)$|layer\\.conf$'
    }
    
    options {
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timestamps()
        disableConcurrentBuilds()
        quietPeriod(5)
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    // Store current commit for next build
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    echo "→ Current commit: ${env.GIT_COMMIT}"
                }
            }
        }
        
        stage('Find Changed Files') {
            steps {
                script {
                    def prev = env.GIT_PREVIOUS_COMMIT ?: env.GIT_PREVIOUS_SUCCESSFUL_COMMIT ?: 'HEAD~1'
                    def current = env.GIT_COMMIT
                    
                    echo "→ Comparing: ${prev} → ${current}"
                    
                    // Handle first run or when prev commit doesn't exist
                    def filesCmd = """
                        if git rev-parse ${prev} >/dev/null 2>&1; then
                            git diff --name-only ${prev} ${current}
                        else
                            git ls-files
                        fi
                    """
                    
                    def allFiles = sh(script: filesCmd, returnStdout: true).trim()
                    env.FILES = sh(script: "echo '${allFiles}' | grep -E '${PATTERN}' || true", returnStdout: true).trim()
                    
                    def count = env.FILES ? env.FILES.split('\n').size() : 0
                    
                    if (count == 0) {
                        echo "ℹ No Yocto files changed"
                        currentBuild.result = 'NOT_BUILT'
                        error("No Yocto files to process")
                    }
                    
                    echo "→ Changed files: ${count}"
                    env.FILES.split('\n').each { file ->
                        if (file?.trim()) echo "  - ${file}"
                    }
                }
            }
        }
        
        stage('Stage Files') {
            steps {
                sh "rm -rf '${TEMP_DOCS}' && mkdir -p '${TEMP_DOCS}'"
                
                script {
                    def staged = 0
                    env.FILES.split('\n').each { file ->
                        if (file?.trim() && fileExists(file)) {
                            def destDir = "${TEMP_DOCS}"
                            // Preserve directory structure for files in subdirectories
                            if (file.contains('/')) {
                                destDir = "${TEMP_DOCS}/" + file.substring(0, file.lastIndexOf('/'))
                                sh "mkdir -p '${destDir}'"
                            }
                            sh "cp '${file}' '${destDir}/'"
                            staged++
                        }
                    }
                    echo "→ Staged: ${staged} files"
                }
            }
        }
        
        stage('Check Vectorstore') {
            steps {
                script {
                    def exists = sh(script: "test -f '${VECTORSTORE}/index.faiss' && echo 'yes' || echo 'no'", returnStdout: true).trim()
                    env.VECTORSTORE_EXISTS = exists
                    
                    if (exists == 'yes') {
                        def size = sh(script: "wc -c < '${VECTORSTORE}/index.faiss'", returnStdout: true).trim()
                        echo "✓ Existing vectorstore found: ${size} bytes"
                        echo "→ Will update existing database"
                    } else {
                        echo "ℹ No vectorstore found"
                        echo "→ Will create new database"
                    }
                }
            }
        }
        
        stage('Update Vectorstore') {
            steps {
                sh """
                    cd ${RAG_BASE}
                    . ${PYTHON_ENV}/bin/activate
                    
                    python3 << 'PYTHON_SCRIPT'
import os
import glob
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import OllamaEmbeddings
from langchain_community.vectorstores import FAISS

# Configuration
staging_dir = '${TEMP_DOCS}'
vectorstore_path = '${VECTORSTORE}'
vectorstore_exists = '${env.VECTORSTORE_EXISTS}' == 'yes'

print(f"Staging directory: {staging_dir}")
print(f"Vectorstore path: {vectorstore_path}")
print(f"Vectorstore exists: {vectorstore_exists}")

# Load new documents recursively
files = []
for root, dirs, filenames in os.walk(staging_dir):
    for filename in filenames:
        files.append(os.path.join(root, filename))

print(f"\\nFound {len(files)} files to process")

if not files:
    print("No files to process")
    exit(0)

documents = []
for file_path in files:
    try:
        loader = TextLoader(file_path, autodetect_encoding=True)
        docs = loader.load()
        for doc in docs:
            doc.metadata['source'] = os.path.relpath(file_path, staging_dir)
            doc.metadata['category'] = 'yocto'
        documents.extend(docs)
        print(f"✓ Loaded: {os.path.relpath(file_path, staging_dir)}")
    except Exception as e:
        print(f"✗ Failed to load {file_path}: {e}")

if not documents:
    print("No documents loaded successfully")
    exit(1)

print(f"\\nTotal documents loaded: {len(documents)}")

# Split documents
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    length_function=len
)
texts = text_splitter.split_documents(documents)
print(f"Split into {len(texts)} chunks")

# Create embeddings
print("\\nInitializing embeddings...")
embeddings = OllamaEmbeddings(model='llama2')

# Update or create vectorstore
if vectorstore_exists:
    print("\\nLoading existing vectorstore...")
    vectorstore = FAISS.load_local(
        vectorstore_path, 
        embeddings, 
        allow_dangerous_deserialization=True
    )
    old_count = vectorstore.index.ntotal
    print(f"Existing vectors: {old_count}")
    
    print("Adding new documents to vectorstore...")
    vectorstore.add_documents(texts)
    new_count = vectorstore.index.ntotal
    print(f"New vectors: {new_count}")
    print(f"Added: {new_count - old_count} vectors")
else:
    print("\\nCreating new vectorstore...")
    vectorstore = FAISS.from_documents(texts, embeddings)
    print(f"Created with {vectorstore.index.ntotal} vectors")

# Save vectorstore
print("\\nSaving vectorstore...")
vectorstore.save_local(vectorstore_path)
print(f"✓ Saved to {vectorstore_path}")
print(f"✓ Total vectors in database: {vectorstore.index.ntotal}")

PYTHON_SCRIPT
                """
            }
        }
        
        stage('Verify Update') {
            steps {
                sh """
                    echo "→ Vectorstore files:"
                    ls -lh ${VECTORSTORE}/*.{faiss,pkl} 2>/dev/null
                    
                    if [ -f "${VECTORSTORE}/index.faiss" ]; then
                        SIZE=\$(wc -c < "${VECTORSTORE}/index.faiss")
                        echo "✓ index.faiss: \${SIZE} bytes"
                    else
                        echo "✗ index.faiss not found!"
                        exit 1
                    fi
                """
            }
        }
        
        stage('Cleanup') {
            steps {
                sh "rm -rf '${TEMP_DOCS}'"
            }
        }
    }
    
    post {
        success {
            script {
                def count = env.FILES?.split('\n')?.size() ?: 0
                def mode = env.VECTORSTORE_EXISTS == 'yes' ? 'Updated' : 'Created'
                echo "✓ ${mode} vectorstore with ${count} new files"
            }
        }
        failure { echo "✗ Vectorstore update failed" }
        always { sh "rm -rf '${TEMP_DOCS}' 2>/dev/null || true" }
    }
}
