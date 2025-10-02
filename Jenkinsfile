pipeline {
  agent any

  tools { nodejs "NodeJS_24" }

  environment {
    PATH       = "/usr/local/bin:${env.PATH}"

    IMAGE      = "realworld-api:${env.BUILD_NUMBER}"

    NET          = "ci-net"
    PG_USER      = "postgres"
    PG_PASS      = "postgres"
    PG_DB        = "realworld_test"
    PG_HOST_CONT = "ci-postgres"
    PG_PORT_CONT = "5432"
    PG_PORT_HOST = "54321"
  }

  stages {
    stage('Preflight') {
      steps {
        sh '''
          set -eux
          whoami
          which docker
          docker version
        '''
      }
    }

    stage('Build') {
      steps {
        checkout scm
        sh '''
          set -eux
          docker build -t "$IMAGE" .
        '''
      }
    }

    stage('Test') {
      steps {
        sh '''
          set -eux

          docker network create "$NET" || true
          docker rm -f "$PG_HOST_CONT" || true
          docker run -d --name "$PG_HOST_CONT" --network "$NET" \
            -e POSTGRES_USER="$PG_USER" \
            -e POSTGRES_PASSWORD="$PG_PASS" \
            -e POSTGRES_DB="$PG_DB" \
            -p "$PG_PORT_HOST:$PG_PORT_CONT" \
            postgres:15

          for i in $(seq 1 40); do
            docker exec "$PG_HOST_CONT" pg_isready -U "$PG_USER" -d "$PG_DB" && break
            sleep 1
          done

          npm ci --include=dev
          export NODE_ENV=test
          export JWT_SECRET=testsecret
          export DATABASE_URL="postgresql://${PG_USER}:${PG_PASS}@localhost:${PG_PORT_HOST}/${PG_DB}?schema=public"

          npx prisma migrate reset --force --skip-seed || npx prisma migrate deploy

          npm test -- --ci --runInBand

          docker rm -f "$PG_HOST_CONT" || true
        '''
      }
    }
  }
}
