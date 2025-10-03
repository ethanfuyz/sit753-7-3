pipeline {
  agent any
  tools { nodejs "NodeJS_24" }

  environment {
    ARTIFACT   = "realworld-api-${env.BUILD_NUMBER}.tar.gz"
    STAGE_DIR  = "build_staging"
    NODE_ENV   = "production"
  }

  stages {
    stage('Build') {
      steps {
        checkout scm
        sh '''
          set -eux

          npm ci --include=dev
          npx prisma generate
          npx nx build api

          rm -rf "$STAGE_DIR" && mkdir -p "$STAGE_DIR"
          cp -r dist node_modules "$STAGE_DIR"/
          [ -d prisma ] && cp -r prisma "$STAGE_DIR"/ || true
          cp package*.json "$STAGE_DIR"/
          tar -czf "$ARTIFACT" -C "$STAGE_DIR" .
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -eux
          
          npm ci --include=dev
          npx nx test api --ci --runInBand --testPathIgnorePatterns=src/tests/services/auth.service.test.ts
        '''
      }
    }

    stage('Quality') {
      environment {
        SCANNER_HOME = tool 'SonarScanner'   
      }
      steps {
        withSonarQubeEnv('MySonar') {        
          sh '''
            set -eux
            npm ci --include=dev

            "$SCANNER_HOME/bin/sonar-scanner"
          '''
        }
      }
    }

    stage('Security') {
      tools { nodejs 'NodeJS_24' } 
      steps {
        withEnv(["PATH+NODE=${tool name: 'NodeJS_24', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'}/bin"]) {
          sh '''
            set -eux

            echo "Running security scan..."
            npm audit --json > audit-report.json || true

            COUNTS=$(node -e "const j=require('./audit-report.json'); const v=(j.metadata&&j.metadata.vulnerabilities)||{}; const out=[v.critical||0, v.high||0, v.moderate||0, v.low||0]; process.stdout.write(out.join(' '))")

            set -- $COUNTS
            CRITICAL=$1; HIGH=$2; MEDIUM=$3; LOW=$4

            echo "Vulnerabilities found -> Critical=$CRITICAL, High=$HIGH, Medium=$MEDIUM, Low=$LOW"

            if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
              echo "❌ Critical or High vulnerabilities detected. Please fix the dependencies."
              exit 1
            else
              echo "✅ No critical or high vulnerabilities detected."
            fi
          '''
        }
      }
    }

  }
}
