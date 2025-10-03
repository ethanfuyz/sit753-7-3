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
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          withEnv(["PATH+NODE=${tool name: 'NodeJS_24', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'}/bin"]) {
            sh '''
              set -euo pipefail

              echo "Running security scan..."
              npm ci --omit=dev
              npm audit --omit=dev --json > audit-report.json || true

              echo "---- npm audit summary ----"
              node -e "
                const j=require('./audit-report.json');
                const m=j.metadata?.vulnerabilities||{};
                console.log('Summary -> Critical=%s, High=%s, Medium=%s, Low=%s', m.critical||0, m.high||0, m.moderate||0, m.low||0);

                const sevRank = {critical:3, high:2, moderate:1, low:0};
                const rows = Object.entries(j.vulnerabilities||{}).map(([name,info])=>{
                  const via = (info.via||[]).map(x=> typeof x==='string'?x:(x?.title||x?.name)).filter(Boolean)[0] || 'N/A';
                  const fix = info.fixAvailable ? (typeof info.fixAvailable==='object' ? (info.fixAvailable.name+'@'+info.fixAvailable.version) : 'Yes') : 'No';
                  const dep = (info.effects||[])[0] || '';
                  return {name, severity: info.severity, issue: via, fix, dep};
                }).sort((a,b)=> (sevRank[b.severity]??-1)-(sevRank[a.severity]??-1));

                if(rows.length){
                  console.log('\\n---- Top issues (first 20) ----');
                  rows.slice(0,20).forEach((r,i)=> console.log(
                    (i+1)+'.', r.name, '|', r.severity.toUpperCase(),
                    '| Issue:', r.issue, '| FixAvailable:', r.fix, r.dep?('| Affected via: '+r.dep):''
                  ));
                } else {
                  console.log('No detailed vulnerabilities listed.');
                }
              " | sed -E 's/^/SECURITY: /'

              COUNTS=$(node -e "const j=require('./audit-report.json'); const m=j.metadata?.vulnerabilities||{}; process.stdout.write([m.critical||0,m.high||0,m.moderate||0,m.low||0].join(' '))")
              set -- $COUNTS; CRITICAL=$1; HIGH=$2
              if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
                echo "SECURITY: ❗ Critical/High vulnerabilities present -> marking stage UNSTABLE"
                exit 2
              else
                echo "SECURITY: ✅ No Critical/High vulnerabilities in production deps."
              fi
            '''
          }
        }
      }
    }
  }
}
