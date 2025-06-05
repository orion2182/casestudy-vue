// Jenkinsfile
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        nodejs 'NodeJS-22'
    }

    environment {
        REMOTE_APP_DIR    = "/var/www/vue"
        REMOTE_USER_HOST  = "target@103.250.10.112"
        PRODUCTION_BRANCH = 'main'
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
                    if (fileExists('.env.production') && !fileExists('.env')) {
                        echo "Menggunakan .env.production yang ada di repo (harap pastikan aman)."
                    } else if (!fileExists('.env')) {
                        sh 'cp .env.example .env'
                        echo "File .env disalin dari .env.example. Harap konfigurasikan untuk produksi di VPS!"
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
                } // Akhir dari script block untuk Stage 6
            } // Akhir dari steps untuk Stage 6
        } // Akhir dari Stage 6

        // --- PERBAIKAN PENEMPATAN STAGE DEBUG ---
        // Stage Debug Branch Info harus sejajar dengan stage lainnya
        stage('Debug Branch Info') { 
            steps {
                script {
                    echo "============================================================"
                    echo "DEBUG Jenkinsfile: Current env.BRANCH_NAME is: '${env.BRANCH_NAME}'"
                    echo "DEBUG Jenkinsfile: Current env.PRODUCTION_BRANCH is: '${env.PRODUCTION_BRANCH}'"
                    echo "DEBUG Jenkinsfile: Comparing them: ${env.BRANCH_NAME == env.PRODUCTION_BRANCH}"
                    echo "============================================================"
                }
            }
        }
        // --- AKHIR DARI PERBAIKAN PENEMPATAN STAGE DEBUG ---

        stage('7. Deploy to VPS') {
            when {
                expression { env.BRANCH_NAME == env.PRODUCTION_BRANCH }
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
                                # php artisan event:cache 
                                # php artisan queue:restart 

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
