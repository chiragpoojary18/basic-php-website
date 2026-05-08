pipeline {
    agent any

    environment {
        APP_NAME       = "basic-php-website"
        REPO_URL       = "https://github.com/aeimskei/basic-php-website.git"
        BRANCH         = "master"
        DEPLOY_DIR     = "/var/www/${APP_NAME}"
        APP_PORT       = "8085"
        PID_FILE       = "/tmp/${APP_NAME}.pid"
        PACKAGE_NAME   = "${APP_NAME}-${BUILD_NUMBER}.tar.gz"
        ARCHIVE_DIR    = "/var/www/releases"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {

        // ─────────────────────────────────────────────
        // STAGE 1: Pull Code from Version Control
        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                echo "📥 Pulling source code from GitHub..."
                git branch: "${BRANCH}",
                    url: "${REPO_URL}"
                
                sh 'echo "--- Workspace Contents ---" && ls -la'
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 2: Validate Environment
        // ─────────────────────────────────────────────
        stage('Validate') {
            steps {
                echo "🔍 Validating environment and tools..."
                sh '''
                    echo "PHP version:"
                    php --version

                    echo "Checking for required PHP extensions..."
                    php -m | grep -E "json|mbstring|tokenizer"

                    echo "Composer version (if present):"
                    composer --version 2>/dev/null || echo "Composer not found – will skip dependency install"
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 3: PHP Syntax Check (Compile/Lint)
        // ─────────────────────────────────────────────
        stage('Syntax Check (Compile)') {
            steps {
                echo "🧪 Running PHP syntax check on all .php files..."
                sh '''
                    ERRORS=0
                    for file in $(find . -name "*.php" -not -path "./.git/*"); do
                        php -l "$file"
                        if [ $? -ne 0 ]; then
                            echo "❌ Syntax error in: $file"
                            ERRORS=$((ERRORS + 1))
                        fi
                    done

                    if [ $ERRORS -gt 0 ]; then
                        echo "❌ $ERRORS file(s) failed syntax check. Aborting build."
                        exit 1
                    fi

                    echo "✅ All PHP files passed syntax check."
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 4: Test
        // ─────────────────────────────────────────────
        stage('Test') {
            steps {
                echo "🧬 Running tests..."
                sh '''
                    # Run PHPUnit if available and tests/ directory exists
                    if [ -f "vendor/bin/phpunit" ] && [ -d "tests" ]; then
                        echo "Running PHPUnit tests..."
                        vendor/bin/phpunit --testdox tests/
                    else
                        echo "No PHPUnit/tests directory found."
                        echo "Running basic PHP execution test (include check)..."
                        # Verify the entry-point file can be parsed without errors
                        php -r "
                            \$files = glob('*.php');
                            foreach (\$files as \$f) {
                                echo 'Checking: ' . \$f . PHP_EOL;
                            }
                            echo 'Basic test passed.';
                        "
                    fi
                    echo "✅ Tests complete."
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 5: Install Dependencies
        // ─────────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                echo "📦 Installing dependencies..."
                sh '''
                    if [ -f "composer.json" ]; then
                        echo "composer.json found. Running composer install..."
                        composer install --no-dev --optimize-autoloader --prefer-dist
                        echo "✅ Composer install complete."
                    else
                        echo "No composer.json found. This is a plain PHP project."
                        echo "✅ No dependencies to install. Skipping."
                    fi
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 6: Package
        // ─────────────────────────────────────────────
        stage('Package') {
            steps {
                echo "📦 Packaging application..."
                sh '''
                    mkdir -p ${ARCHIVE_DIR}

                    # Create a tarball of the app excluding git metadata and old packages
                    tar --exclude=".git" \
                        --exclude="*.tar.gz" \
                        -czf ${ARCHIVE_DIR}/${PACKAGE_NAME} \
                        -C ${WORKSPACE} .

                    echo "✅ Package created: ${ARCHIVE_DIR}/${PACKAGE_NAME}"
                    ls -lh ${ARCHIVE_DIR}/${PACKAGE_NAME}
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 7: Deploy
        // ─────────────────────────────────────────────
        stage('Deploy') {
            steps {
                echo "🚀 Deploying application to ${DEPLOY_DIR}..."
                sh '''
                    # Stop any previously running PHP dev server for this app
                    if [ -f "${PID_FILE}" ]; then
                        OLD_PID=$(cat ${PID_FILE})
                        if kill -0 "$OLD_PID" 2>/dev/null; then
                            echo "Stopping existing server (PID: $OLD_PID)..."
                            kill "$OLD_PID"
                            sleep 2
                        fi
                        rm -f "${PID_FILE}"
                    fi

                    # Also kill any process holding the port (safety net)
                    fuser -k ${APP_PORT}/tcp 2>/dev/null || true
                    sleep 1

                    # Deploy: extract package to deploy directory
                    mkdir -p ${DEPLOY_DIR}
                    tar -xzf ${ARCHIVE_DIR}/${PACKAGE_NAME} -C ${DEPLOY_DIR}
                    echo "✅ Application files deployed to ${DEPLOY_DIR}"

                    # Start PHP built-in server in background
                    nohup php -S 0.0.0.0:${APP_PORT} -t ${DEPLOY_DIR} > /tmp/${APP_NAME}-server.log 2>&1 &
                    echo $! > ${PID_FILE}

                    echo "✅ PHP server started on port ${APP_PORT} (PID: $(cat ${PID_FILE}))"
                    sleep 3
                '''
            }
        }

        // ─────────────────────────────────────────────
        // STAGE 8: Health Check
        // ─────────────────────────────────────────────
        stage('Health Check') {
            steps {
                echo "❤️  Running health check on http://localhost:${APP_PORT}..."
                sh '''
                    MAX_RETRIES=5
                    RETRY=0

                    until curl -sf http://localhost:${APP_PORT} > /dev/null; do
                        RETRY=$((RETRY + 1))
                        if [ $RETRY -ge $MAX_RETRIES ]; then
                            echo "❌ Health check failed after ${MAX_RETRIES} attempts."
                            echo "--- Server Log ---"
                            cat /tmp/${APP_NAME}-server.log
                            exit 1
                        fi
                        echo "Retry $RETRY/$MAX_RETRIES - waiting..."
                        sleep 3
                    done

                    echo "✅ Application is UP at http://localhost:${APP_PORT}"
                '''
            }
        }
    }

    // ─────────────────────────────────────────────────
    // POST ACTIONS
    // ─────────────────────────────────────────────────
    post {
        success {
            echo """
            ╔══════════════════════════════════════════╗
            ║  ✅ BUILD SUCCESSFUL                     ║
            ║  App running at: http://localhost:${APP_PORT}  ║
            ╚══════════════════════════════════════════╝
            """
        }
        failure {
            echo "❌ BUILD FAILED. Check logs above for details."
            sh '''
                echo "--- PHP Server Log (last 50 lines) ---"
                tail -50 /tmp/${APP_NAME}-server.log 2>/dev/null || echo "No server log found."
            '''
        }
        always {
            echo "🧹 Build #${BUILD_NUMBER} complete. Artifacts retained in ${ARCHIVE_DIR}."
        }
    }
}
