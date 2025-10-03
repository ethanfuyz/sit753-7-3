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
      steps {
        sh '''
          set -eux

          npm ci --include=dev
          echo "Running security scan..."

          if npm audit --audit-level=high --json > audit-report.json; then
            node -e "const j=require('./audit-report.json');const v=j.metadata?.vulnerabilities||{};console.log(`Vulnerabilities found: Critical=${v.critical||0}, High=${v.high||0}, Medium=${v.moderate||0}, Low=${v.low||0}`)"
            echo "✅ No Critical/High vulnerabilities detected."
          else
            node -e "const j=require('./audit-report.json');const v=j.metadata?.vulnerabilities||{};console.log(`Vulnerabilities found: Critical=${v.critical||0}, High=${v.high||0}, Medium=${v.moderate||0}, Low=${v.low||0}`)"
            echo "❌ Critical or High vulnerabilities detected. Please fix the dependencies."
            exit 1
          fi
        '''
      }
    }
  }
}
