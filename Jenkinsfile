pipeline {
    agent any

    // Параметры job
    parameters {
        string(
            name: 'RemoteHostVm',
            defaultValue: '192.168.31.120',
            description: 'IP или hostname удалённого сервера'
        )
    }

    stages {

        stage('1. Init ssh-connection and check Docker') {
            steps {
                script {

                    // Глобальная переменная подключения
                    RemoteConnectionSsh = null

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'ssh-elma1-pass-and-login',
                            usernameVariable: 'SshUser',
                            passwordVariable: 'SshPassword'
                        )
                    ]) {

                        // SSH подключение
                        RemoteConnectionSsh = [
                            name: 'elma',
                            host: params.RemoteHostVm,
                            user: SshUser,
                            password: SshPassword,
                            allowAnyHosts: true
                        ]

                        echo "----------SSH connection success----------"
                        echo "Host: ${params.RemoteHostVm}"
                        echo "User: ${SshUser}"

                        echo "----------Docker containers----------"
                        sshCommand remote: RemoteConnectionSsh, command: "sudo docker ps"
                    }
                }
            }
        }

        stage('2. Get MongoDB and PostgreSQL credentials') {
            steps {
                script {

                    echo "----------Get MongoDB credentials----------"

                    def mongoUrl = sshCommand(
                        remote: RemoteConnectionSsh,
                        command: """
sudo docker exec elma365 kubectl get secret elma365-db-connections \
-o jsonpath='{.data.MONGO_URL}' | base64 -d
""",
                        returnStdout: true
                    ).trim()

                    echo "MongoDB URL: ${mongoUrl}"

                    // Извлекаем пароль MongoDB
                    env.MONGO_PASS = (mongoUrl =~ /\\/\\/[^:]+:([^@]+)@/)[0][1]

                    echo "MongoDB Password extracted"

                    echo "----------Get PostgreSQL credentials----------"

                    def psqlUrl = sshCommand(
                        remote: RemoteConnectionSsh,
                        command: """
sudo docker exec elma365 kubectl get secret elma365-db-connections \
-o jsonpath='{.data.PSQL_URL}' | base64 -d
""",
                        returnStdout: true
                    ).trim()

                    echo "PostgreSQL URL: ${psqlUrl}"

                    // Извлекаем данные PostgreSQL
                    env.PG_PASS = (psqlUrl =~ /:\\/\\/([^:]+):([^@]+)@/)[0][2]
                    env.PG_DB = (psqlUrl =~ /@[^\\/]+\\/([^?]+)/)[0][1]

                    echo "PostgreSQL Password extracted"
                    echo "PostgreSQL Database: ${env.PG_DB}"
                }
            }
        }

        stage('3. Remove old DBGate container if exists') {
            steps {
                script {

                    echo "----------Search DBGate container----------"

                    def containerExists = sshCommand(
                        remote: RemoteConnectionSsh,
                        command: "sudo docker ps -a --filter 'name=dbgate' --format '{{.Names}}'",
                        returnStdout: true
                    ).trim()

                    if (containerExists) {

                        echo "----------DBGate container found----------"
                        echo "----------Stopping container----------"

                        sshCommand(
                            remote: RemoteConnectionSsh,
                            command: "sudo docker stop dbgate || true"
                        )

                        echo "----------Removing container----------"

                        sshCommand(
                            remote: RemoteConnectionSsh,
                            command: "sudo docker rm dbgate || true"
                        )

                        echo "----------DBGate removed----------"

                    } else {

                        echo "----------DBGate container not found----------"
                    }

                    echo "----------Docker containers----------"

                    sshCommand(
                        remote: RemoteConnectionSsh,
                        command: "sudo docker ps -a"
                    )
                }
            }
        }

        stage('4. Run DBGate container') {
            steps {
                script {

                    echo "----------Run DBGate container----------"

                    sshCommand(
                        remote: RemoteConnectionSsh,
                        command: """
sudo docker run -d --name dbgate \\
    -e LABEL_mongo=MongoLocal \\
    -e ENGINE_mongo=mongo@dbgate-plugin-mongo \\
    -e URL_mongo="mongodb://elma365:${env.MONGO_PASS}@${params.RemoteHostVm}:27017/elma365?ssl=false&directConnection=true" \\
    -e LABEL_postgres=PostgresLocal \\
    -e ENGINE_postgres=postgres@dbgate-plugin-postgres \\
    -e USER_postgres=postgres \\
    -e OPTIONS_postgres="sslmode=disable" \\
    -e CONNECTIONS="mongo,postgres" \\
    -e SERVER_postgres=${params.RemoteHostVm} \\
    -e PORT_postgres=5432 \\
    -e PASSWORD_postgres=${env.PG_PASS} \\
    -e DATABASE_postgres=${env.PG_DB} \\
    -p 3000:3000 \\
    --restart unless-stopped \\
    dbgate/dbgate
"""
                    )

                    echo "----------DBGate started successfully----------"

                    sshCommand(
                        remote: RemoteConnectionSsh,
                        command: "sudo docker ps | grep dbgate"
                    )

                    echo "----------DBGate URL----------"
                    echo "http://${params.RemoteHostVm}:3000"
                }
            }
        }
    }
}
