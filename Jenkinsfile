// Jenkinsfile (Versi dengan Perbaikan Sintaks dan Logika 'when')
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        nodejs 'NodeJS-22'
    }

    environment {
        REMOTE_APP_DIR    = "/var/www/vue"
        REMOTE_USER_HOST  = "target@103.250.10.112"
        PRODUCTION_BRANCH = 'main' // Pastikan ini nama branch produksi Anda
        // Kita tidak perlu ACTUAL_BRANCH_NAME di sini lagi karena logikanya ada di 'when'
    }

    stages {
        stage('1. Clone Repository') {
            steps {
                script {
                    cleanWs()
                    checkout scm
                    sh 'git log -1'
                }
            }
        }

        stage('2. Setup Laravel .env File') {
            steps {
                script {
                    if (!fileExists('.env')) {
                        sh 'cp .env.example .env'
                        echo "File .env disalin dari .env.example. Ini HANYA untuk build di Jenkins. Pastikan .env produksi Anda aman di VPS!"
                    }
                }
            }
        }

        stage('3. Install PHP Dependencies (Composer)') {
            steps {
                sh 'composer install --no-dev --optimize-autoloader --no-interaction --no-progress'
            }
        }

        stage('4. Install Node.js Dependencies & Build Vue.js Assets') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }

        // stage('5. Run Tests') { /* ... */ }

        stage('6. Prepare Deployment Package') {
            steps {
                script {
                    echo "Membuat arsip deployment..."
                    sh "tar -czf /tmp/deployment_pkg_temp.tar.gz --exclude='./.git' --exclude='./node_modules' --exclude='./.env' --exclude='./Jenkinsfile' --exclude='./storage/logs/*' --exclude='./storage/framework/sessions/*' --exclude='./storage/framework/cache/*' --exclude='./storage/framework/views/*' --exclude='./public/storage' -C ${env.WORKSPACE} ."
                    echo "Memindahkan arsip ke workspace..."
                    sh "mv /tmp/deployment_pkg_temp.tar.gz ${env.WORKSPACE}/deployment.tar.gz"
                    echo "Arsip deployment.tar.gz berhasil dibuat di workspace."
                }
            }
        }

        // Tahap ini sekarang MURNI untuk logging/debugging nilai branch
        stage('Log Branch Information') {
            steps {
                script {
                    def determinedBranchForLog = 'unknown'
                    if (env.BRANCH_NAME && env.BRANCH_NAME != 'null' && env.BRANCH_NAME.trim() != '') {
                        determinedBranchForLog = env.BRANCH_NAME.trim()
                    } else if (env.GIT_BRANCH && env.GIT_BRANCH != 'null' && env.GIT_BRANCH.trim() != '') {
                        def parts = env.GIT_BRANCH.split('/')
                        determinedBranchForLog = parts[parts.size() - 1].trim()
                    }

                    echo "============================================================"
                    echo "LOGGING: env.BRANCH_NAME raw is: '${env.BRANCH_NAME}'"
                    echo "LOGGING: env.GIT_BRANCH raw is: '${env.GIT_BRANCH}'"
                    echo "LOGGING: Branch determined for logging purposes: '${determinedBranchForLog}'"
                    echo "LOGGING: env.PRODUCTION_BRANCH is: '${env.PRODUCTION_BRANCH}'"
                    // Perbandingan ini dicatat di log, tapi logika sebenarnya ada di 'when' tahap deploy
                    echo "LOGGING: Comparison (for log only): ${determinedBranchForLog == env.PRODUCTION_BRANCH}"
                    echo "============================================================"
                }
            }
        }

        stage('7. Deploy to VPS') {
            when {
                expression {
                    // Logika penentuan branch sekarang ada DI DALAM expression ini
                    def effectiveBranch = 'unknown'
                    if (env.BRANCH_NAME && env.BRANCH_NAME != 'null' && env.BRANCH_NAME.trim() != '') {
                        effectiveBranch = env.BRANCH_NAME.trim()
                    } else if (env.GIT_BRANCH && env.GIT_BRANCH != 'null' && env.GIT_BRANCH.trim() != '') {
                        def gitBranchParts = env.GIT_BRANCH.split('/')
                        effectiveBranch = gitBranchParts[gitBranchParts.size() - 1].trim()
                    }
                    // Kondisi aktual untuk menjalankan tahap ini
                    // 'return' bersifat implisit untuk baris terakhir di blok expression Groovy
                    effectiveBranch == env.PRODUCTION_BRANCH && effectiveBranch != 'unknown'
                }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'vps-target-deploy-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')]) {
                    script {
                        def remoteHost = env.REMOTE_USER_HOST.split('@')[1]
                        def remoteUser = env.REMOTE_USER_HOST.split('@')[0]

                        sh "scp -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null deployment.tar.gz ${remoteUser}@${remoteHost}:/tmp/deployment.tar.gz"
                        sh """
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${remoteUser}@${remoteHost} 'bash -s' <<EOF
                                set -e 
                                echo "Memulai proses deployment di ${env.REMOTE_APP_DIR}..."
                                cd ${env.REMOTE_APP_DIR}
                                php artisan down || echo "Gagal mengaktifkan maintenance mode atau sudah aktif."
                                echo "Menghapus file lama..."
                                find . -maxdepth 1 ! -name '.env' ! -name 'storage' ! -name '.git' -exec rm -rf {} \\; || echo "Tidak ada file lama untuk dihapus atau error saat menghapus."
                                echo "Mengekstrak arsip baru..."
                                tar -xzf /tmp/deployment.tar.gz -C ${env.REMOTE_APP_DIR}
                                echo "Menghapus arsip di VPS..."
                                rm /tmp/deployment.tar.gz
                                echo "Mengatur izin..."
                                sudo chmod -R ug+w storage bootstrap/cache
                                echo "Menjalankan perintah pasca-deployment Laravel..."
                                php artisan storage:link || echo "Gagal membuat storage link atau sudah ada."
                                php artisan migrate --force
                                php artisan optimize:clear
                                php artisan config:cache
                                php artisan route:cache
                                php artisan view:cache
                                echo "Merestart PHP-FPM..."
                                sudo systemctl restart php8.3-fpm
                                php artisan up || echo "Gagal menonaktifkan maintenance mode."
                                echo "DEPLOYMENT SELESAI! Aplikasi di ${env.REMOTE_APP_DIR} telah diperbarui."
                            EOF
                        """
                    }
                }
            }
        }
    } // Akhir dari blok 'stages' utama

    post {
        always {
            echo 'Pipeline selesai.'
            deleteDir()
        }
        success {
            echo 'Pipeline berhasil!'
        }
        failure {
            echo 'Pipeline GAGAL!'
        }
    }
}
