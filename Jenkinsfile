// Jenkinsfile (Versi Lengkap dan Diperbarui)
pipeline {
    agent any // Menjalankan di agent Jenkins manapun yang tersedia

    tools {
        // PASTIKAN NAMA INI SESUAI DENGAN YANG DIKONFIGURASI DI JENKINS > GLOBAL TOOL CONFIGURATION
        jdk 'JDK17'
        nodejs 'NodeJS-22'
    }

    environment {
        // Variabel Lingkungan untuk Pipeline
        REMOTE_APP_DIR     = "/var/www/vue"             // Direktori root aplikasi di VPS Anda
        REMOTE_USER_HOST   = "target@103.250.10.112"  // Ganti dengan user dan IP VPS Anda
        PRODUCTION_BRANCH  = 'main'                     // GANTI DENGAN NAMA BRANCH PRODUKSI ANDA JIKA BUKAN 'main'
        ACTUAL_BRANCH_NAME = ''                         // Variabel untuk menyimpan nama branch yang terdeteksi

        // Opsional: ID Credential untuk file .env.production jika disimpan di Jenkins
        // PROD_ENV_CREDENTIAL_ID = 'laravel-production-env-file' 
    }

    stages {
        stage('1. Clone Repository') {
            steps {
                script {
                    cleanWs() // Bersihkan workspace sebelum clone
                    checkout scm // Checkout kode dari Git (SCM yang dikonfigurasi di job)
                    sh 'git log -1'
                }
            }
        }

        stage('2. Setup Laravel .env File') {
            steps {
                script {
                    // Strategi .env:
                    // Untuk produksi, sangat disarankan memiliki file .env yang aman di server dan tidak di-commit,
                    // atau gunakan Jenkins credentials (secret file/text) untuk menyuntikkannya saat deployment.
                    // Hindari menyimpan kredensial asli di .env.example atau .env.production di dalam repo.
                    if (!fileExists('.env')) {
                        sh 'cp .env.example .env'
                        echo "File .env disalin dari .env.example. Ini HANYA untuk build di Jenkins. Pastikan .env produksi Anda aman di VPS!"
                        // Jika perlu, modifikasi .env untuk build (misalnya, database testing, dll.)
                        // Contoh: sh "sed -i 's/APP_ENV=local/APP_ENV=testing/' .env"
                    }
                    // Contoh jika .env.production disimpan sebagai "Secret file" di Jenkins dan ingin digunakan di workspace build:
                    // withCredentials([file(credentialsId: env.PROD_ENV_CREDENTIAL_ID, variable: 'PROD_ENV_FILE_PATH')]) {
                    //    sh "cp ${PROD_ENV_FILE_PATH} .env" 
                    //    echo "Menggunakan .env dari Jenkins Credentials ID: ${env.PROD_ENV_CREDENTIAL_ID}"
                    // }
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
                sh 'npm run build' // Atau 'yarn build' jika Anda menggunakan Yarn
            }
        }

        // stage('5. Run Tests') { // Opsional, tapi sangat direkomendasikan
        //     steps {
        //         // Pastikan .env sudah dikonfigurasi untuk database testing jika perlu
        //         // sh 'php artisan migrate --env=testing --force' 
        //         sh 'php artisan test'
        //     }
        // }

        stage('6. Prepare Deployment Package') {
            steps {
                script {
                    echo "Membuat arsip deployment..."
                    // Membuat arsip di /tmp lalu memindahkannya ke workspace
                    sh "tar -czf /tmp/deployment_pkg_temp.tar.gz --exclude='./.git' --exclude='./node_modules' --exclude='./.env' --exclude='./Jenkinsfile' --exclude='./storage/logs/*' --exclude='./storage/framework/sessions/*' --exclude='./storage/framework/cache/*' --exclude='./storage/framework/views/*' --exclude='./public/storage' -C ${env.WORKSPACE} ."
                    
                    echo "Memindahkan arsip ke workspace..."
                    sh "mv /tmp/deployment_pkg_temp.tar.gz ${env.WORKSPACE}/deployment.tar.gz"
                    
                    echo "Arsip deployment.tar.gz berhasil dibuat di workspace."
                }
            }
        }

        // TAHAP DEBUG DAN PENENTUAN BRANCH YANG DIPERBARUI
        stage('Determine Branch Name & Debug') {
            steps {
                script {
                    // Coba dapatkan nama branch
                    if (env.BRANCH_NAME && env.BRANCH_NAME != 'null' && env.BRANCH_NAME.trim() != '') {
                        env.ACTUAL_BRANCH_NAME = env.BRANCH_NAME.trim()
                    } else if (env.GIT_BRANCH && env.GIT_BRANCH != 'null' && env.GIT_BRANCH.trim() != '') {
                        // env.GIT_BRANCH mungkin 'origin/main', jadi kita ambil bagian terakhir setelah '/'
                        def parts = env.GIT_BRANCH.split('/')
                        env.ACTUAL_BRANCH_NAME = parts[parts.size() - 1].trim()
                    } else {
                        echo "PERINGATAN: Tidak bisa menentukan nama branch dari env.BRANCH_NAME atau env.GIT_BRANCH."
                        env.ACTUAL_BRANCH_NAME = 'unknown' // Atau pertimbangkan `error()` jika ini kondisi kritis
                    }

                    echo "============================================================"
                    echo "DEBUG Jenkinsfile: env.BRANCH_NAME raw is: '${env.BRANCH_NAME}'"
                    echo "DEBUG Jenkinsfile: env.GIT_BRANCH raw is: '${env.GIT_BRANCH}'"
                    echo "DEBUG Jenkinsfile: Determined ACTUAL_BRANCH_NAME is: '${env.ACTUAL_BRANCH_NAME}'"
                    echo "DEBUG Jenkinsfile: env.PRODUCTION_BRANCH is: '${env.PRODUCTION_BRANCH}'"
                    echo "DEBUG Jenkinsfile: Comparing them: ${env.ACTUAL_BRANCH_NAME == env.PRODUCTION_BRANCH}"
                    echo "============================================================"
                }
            }
        }

        stage('7. Deploy to VPS') {
            when {
                // Gunakan ACTUAL_BRANCH_NAME yang sudah ditentukan untuk perbandingan
                // dan pastikan bukan 'unknown'
                expression { env.ACTUAL_BRANCH_NAME == env.PRODUCTION_BRANCH && env.ACTUAL_BRANCH_NAME != 'unknown' }
            }
            steps {
                // PASTIKAN CREDENTIALS ID INI SESUAI DENGAN YANG ANDA BUAT DI JENKINS
                withCredentials([sshUserPrivateKey(credentialsId: 'vps-target-deploy-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')]) {
                    script {
                        def remoteHost = env.REMOTE_USER_HOST.split('@')[1]
                        def remoteUser = env.REMOTE_USER_HOST.split('@')[0] // Ini akan menjadi 'target'

                        echo "Mengirim arsip deployment ke ${remoteUser}@${remoteHost}..."
                        sh "scp -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null deployment.tar.gz ${remoteUser}@${remoteHost}:/tmp/deployment.tar.gz"

                        echo "Menjalankan skrip deployment di VPS..."
                        sh """
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${remoteUser}@${remoteHost} 'bash -s' <<EOF
                                # Hentikan skrip jika ada error
                                set -e 

                                echo "Memulai proses deployment di direktori: ${env.REMOTE_APP_DIR}"
                                cd ${env.REMOTE_APP_DIR}

                                echo "Mengaktifkan maintenance mode..."
                                php artisan down || echo "Gagal mengaktifkan maintenance mode atau sudah aktif."

                                echo "Menghapus file lama (kecuali .env dan storage)..."
                                # Berhati-hati dengan perintah find -exec rm. Pastikan Anda tahu apa yang dilakukannya.
                                find . -maxdepth 1 ! -name '.env' ! -name 'storage' ! -name '.gitkeep' -exec rm -rf {} \\; || echo "Tidak ada file lama untuk dihapus atau error saat menghapus."
                                
                                echo "Mengekstrak arsip baru..."
                                tar -xzf /tmp/deployment.tar.gz -C ${env.REMOTE_APP_DIR}

                                echo "Menghapus arsip sementara di VPS..."
                                rm /tmp/deployment.tar.gz

                                echo "Mengatur izin untuk storage dan bootstrap/cache..."
                                # Pastikan user 'target' memiliki hak sudo untuk chmod ini jika diperlukan,
                                # atau direktori ini sudah dimiliki oleh 'target:www-data' dengan izin yang sesuai.
                                sudo chmod -R ug+w storage bootstrap/cache

                                echo "Menjalankan perintah pasca-deployment Laravel..."
                                php artisan storage:link || echo "Gagal membuat storage link atau sudah ada."
                                
                                echo "Menjalankan migrasi database..."
                                php artisan migrate --force

                                echo "Mengoptimalkan aplikasi..."
                                php artisan optimize:clear
                                php artisan config:cache
                                php artisan route:cache
                                php artisan view:cache
                                # php artisan event:cache  # Aktifkan jika Anda menggunakan event discovery
                                # php artisan queue:restart # Aktifkan jika Anda menggunakan Laravel Queues

                                echo "Merestart PHP-FPM..."
                                # Pastikan user 'target' memiliki hak sudo tanpa password untuk perintah ini.
                                sudo systemctl restart php8.3-fpm

                                echo "Menonaktifkan maintenance mode..."
                                php artisan up || echo "Gagal menonaktifkan maintenance mode."

                                echo "DEPLOYMENT SELESAI! Aplikasi di ${env.REMOTE_APP_DIR} telah diperbarui."
                            EOF
                        """
                        echo "Skrip deployment di VPS selesai dijalankan."
                    }
                }
            }
        }
    } // Akhir dari blok 'stages' utama

    post {
        always {
            echo 'Pipeline selesai.'
            deleteDir() // Membersihkan workspace Jenkins setelah build
        }
        success {
            echo 'Pipeline berhasil!'
            // Tambahkan notifikasi (misal: email, Slack)
        }
        failure {
            echo 'Pipeline GAGAL!'
            // Tambahkan notifikasi kegagalan
        }
    }
}
