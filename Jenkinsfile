// Jenkinsfile
pipeline {
    agent any // Menjalankan di agent Jenkins manapun yang tersedia

    tools {
        jdk 'JDK17'         // Nama JDK yang dikonfigurasi di Jenkins Global Tools
        nodejs 'NodeJS-22'  // Nama NodeJS yang dikonfigurasi di Jenkins Global Tools
    }

    environment {
        // Variabel Lingkungan untuk Pipeline
        REMOTE_APP_DIR    = "/var/www/vue" // Direktori root aplikasi di VPS
        REMOTE_USER_HOST  = "target@103.250.10.112" // User dan IP VPS Anda
        PRODUCTION_BRANCH = 'main' // Branch yang akan di-deploy ke produksi
        // Anda bisa menambahkan DB_CONNECTION, APP_KEY, dll di sini atau menggunakan Jenkins credentials
        // Contoh jika .env.production dikelola sebagai secret file di Jenkins:
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
                    // 1. Jika Anda punya .env.production yang aman di server, skip langkah ini di Jenkins.
                    // 2. Atau, gunakan Jenkins credentials untuk menyuntikkan .env.
                    // Di sini kita contohkan menyalin .env.example jika .env tidak ada.
                    // Untuk produksi, Anda HARUS punya .env yang aman.
                    if (fileExists('.env.production') && !fileExists('.env')) {
                        // Jika Anda menyimpan .env.production di repo (TIDAK DIREKOMENDASIKAN untuk kredensial asli)
                        // sh 'cp .env.production .env'
                        echo "Menggunakan .env.production yang ada di repo (harap pastikan aman)."
                    } else if (!fileExists('.env')) {
                        sh 'cp .env.example .env'
                        echo "File .env disalin dari .env.example. Harap konfigurasikan untuk produksi di VPS!"
                        // Anda mungkin perlu mengganti beberapa nilai di .env menggunakan sed
                        // sh "sed -i 's/APP_ENV=local/APP_ENV=production/' .env"
                        // sh "sed -i 's/APP_DEBUG=true/APP_DEBUG=false/' .env"
                    }

                    // Contoh jika .env.production disimpan sebagai "Secret file" di Jenkins:
                    // withCredentials([file(credentialsId: env.PROD_ENV_CREDENTIAL_ID, variable: 'PROD_ENV_CONTENT')]) {
                    //    sh "cp ${PROD_ENV_CONTENT} .env" // Atau .env.production jika Anda ingin menyalinnya ke server
                    // }
                }
            }
        }

        stage('3. Install PHP Dependencies (Composer)') {
            steps {
                // Pastikan composer ada di PATH server Jenkins atau gunakan path absolut
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
        //         sh 'php artisan test'
        //     }
        // }

        stage('6. Prepare Deployment Package') {
            steps {
                script {
                    echo "Membuat arsip deployment..."
                    // Membuat arsip di /tmp lalu memindahkannya ke workspace
                    // Menggunakan -C ${env.WORKSPACE} untuk memastikan tar beroperasi pada direktori workspace
                    // dan '.' setelah -C mengacu pada direktori workspace tersebut.
                    sh "tar -czf /tmp/deployment_pkg_temp.tar.gz --exclude='./.git' --exclude='./node_modules' --exclude='./.env' --exclude='./Jenkinsfile' --exclude='./storage/logs/*' --exclude='./storage/framework/sessions/*' --exclude='./storage/framework/cache/*' --exclude='./storage/framework/views/*' --exclude='./public/storage' -C ${env.WORKSPACE} ."
                    
                    echo "Memindahkan arsip ke workspace..."
                    sh "mv /tmp/deployment_pkg_temp.tar.gz ${env.WORKSPACE}/deployment.tar.gz"
                    
                    echo "Arsip deployment.tar.gz berhasil dibuat di workspace."
                }
            }
        }

        stage('7. Deploy to VPS') {
            // Hanya deploy jika build berasal dari branch produksi yang ditentukan
            when {
                branch env.PRODUCTION_BRANCH
            }
            steps {
                // Gunakan credential SSH yang sudah disetup di Jenkins
                withCredentials([sshUserPrivateKey(credentialsId: 'vps-target-deploy-key', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')]) {
                    script {
                        def remoteHost = env.REMOTE_USER_HOST.split('@')[1]
                        def remoteUser = env.REMOTE_USER_HOST.split('@')[0] // Seharusnya 'target'

                        // 1. Kirim arsip ke VPS
                        sh "scp -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null deployment.tar.gz ${remoteUser}@${remoteHost}:/tmp/deployment.tar.gz"

                        // 2. Eksekusi perintah di VPS
                        sh """
                            ssh -i ${SSH_KEY_FILE} -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${remoteUser}@${remoteHost} 'bash -s' <<EOF
                                # Hentikan skrip jika ada error
                                set -e 

                                echo "Memulai proses deployment di ${env.REMOTE_APP_DIR}..."

                                # Pindah ke direktori aplikasi
                                cd ${env.REMOTE_APP_DIR}

                                # (Opsional) Aktifkan Maintenance Mode Laravel
                                # Pastikan user 'target' bisa menjalankan php artisan atau punya hak sudo untuknya jika perlu
                                php artisan down || echo "Gagal mengaktifkan maintenance mode atau sudah aktif."

                                echo "Menghapus file lama (hati-hati, pastikan backup ada jika perlu)..."
                                # Hapus konten lama KECUALI .env dan storage (jika dikelola terpisah)
                                # Strategi ini bisa disesuaikan, misal menggunakan rsync atau blue/green deployment
                                find . -maxdepth 1 ! -name '.env' ! -name 'storage' ! -name '.git' -exec rm -rf {} \\; || echo "Tidak ada file lama untuk dihapus atau error saat menghapus."
                                
                                echo "Mengekstrak arsip baru..."
                                tar -xzf /tmp/deployment.tar.gz -C ${env.REMOTE_APP_DIR}

                                echo "Menghapus arsip di VPS..."
                                rm /tmp/deployment.tar.gz

                                echo "Mengatur izin..."
                                # Kepemilikan sudah diatur saat setup VPS (target:www-data)
                                # Pastikan izin untuk storage dan bootstrap cache bisa ditulis oleh web server (www-data)
                                sudo chmod -R ug+w storage bootstrap/cache

                                echo "Menjalankan perintah pasca-deployment Laravel..."
                                # Link storage (jika belum ada dan tidak masuk dalam arsip)
                                # Mungkin perlu sudo jika direktori public dimiliki root
                                php artisan storage:link || echo "Gagal membuat storage link atau sudah ada."
                                
                                # Jalankan migrasi database (opsi --force untuk produksi)
                                php artisan migrate --force

                                # Clear cache dan optimasi lainnya
                                php artisan optimize:clear
                                php artisan config:cache
                                php artisan route:cache
                                php artisan view:cache
                                # php artisan event:cache # Jika menggunakan event discovery
                                # php artisan queue:restart # Jika menggunakan antrian

                                echo "Merestart PHP-FPM..."
                                # Perintah ini membutuhkan hak sudo untuk user 'target' di VPS (konfigurasi di /etc/sudoers)
                                sudo systemctl restart php8.3-fpm

                                # (Opsional) Nonaktifkan Maintenance Mode
                                php artisan up || echo "Gagal menonaktifkan maintenance mode."

                                echo "DEPLOYMENT SELESAI! Aplikasi di ${env.REMOTE_APP_DIR} telah diperbarui."
                            EOF
                        """
                    }
                }
            }
        }
    }

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
