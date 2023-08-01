pipeline {
    agent any
    
    parameters {
        string(name: 'ROLE_NAME', description: 'New Role name')
        string(name: 'NEW_USER', description: 'New user name')
        choice(name: 'USER_PERMISSION', choices: ['Read', 'Administer'], description: 'New user permission')
        password(name: 'NEW_USER_PASSWORD1', description: 'New user password')
        password(name: 'NEW_USER_PASSWORD2', description: 'Repeat new user password')
        string(name: 'NEW_USER_EMAIL', description: 'Email address of the new user')
    }
    environment {
        // ROLE_NAME = 'Developers'
        JENKINS_URL = 'http://localhost:8080/'
        JENKINS_USER = 'admin'
        // JENKINS_API_TOKEN = 'JENKINS_API_TOKEN'
        //JENKINS_CRUMB = 'YOUR_CRUMB_VALUE'
    }
    stages {
        stage('Install jq') {
            steps {
                // Install jq using package manager (e.g., apt, yum, brew, etc.)
                sh 'apt-get update && apt-get install -y jq' // Modify this based on your package manager
            }
        }
        stage('Create Role') {
            steps {
                script {
                    def roleName = env.ROLE_NAME
                    def userPermission = env.USER_PERMISSION
                    // Fetch existing roles using a GET request to the Role-Based Strategy API
                    def existingRolesResponse
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        existingRolesResponse = sh(
                            returnStdout: true,
                            script: "curl -s -X GET -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}role-strategy/strategy/getAllRoles'"
                        )
                    }
        
                    // Parse JSON using jq to get the list of existing role names
                    def existingRoleNames = sh(
                        returnStdout: true,
                        script: "echo '${existingRolesResponse}' | jq -r 'keys[]'"
                    ).trim().split("\\s+")
        
                    // Check if the new role name already exists
                    while (existingRoleNames.contains(roleName)) {
                        echo "Role '${roleName}' already exists."
                        roleName = input(
                            id: 'roleInput',
                            message: 'Please enter a new role name:',
                            parameters: [
                                string(defaultValue: "", description: 'New role name', name: 'ROLE_NAME')
                            ]
                        )
                        if (!roleName) {
                            error("No new role name provided. Pipeline aborted.")
                        }
        
                        // Update the existing roles list
                        existingRoleNames = sh(
                            returnStdout: true,
                            script: "echo '${existingRolesResponse}' | jq -r 'keys[]'"
                        ).trim().split("\\s+")
                    }
        
                    // Using 'withCredentials' to securely pass the JENKINS_API_TOKEN
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        def maskedToken = sh(script: 'echo $JENKINS_API_TOKEN', returnStdout: true).trim()
                        echo "Masked JENKINS_API_TOKEN: ${maskedToken}"
                        def response = sh(
                            returnStdout: true,
                            script: "curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' -d 'type=globalRoles&roleName=${roleName}&pattern=&permissions=hudson.model.Item.Build&permissionIds=hudson.model.Hudson.${userPermission}&overwrite=true' -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}role-strategy/strategy/addRole'"
                        )
                    echo "Response from Create Role API: ${response.trim()}"
                    }
                }            
                echo "Finished the 'Create Role' stage."
            }
        }

    
        stage('Create Jenkins User') {
            steps {
                script {
                    def newUserName = env.NEW_USER
                    def newUserPassword1 = env.NEW_USER_PASSWORD1
                    def newUserPassword2 = env.NEW_USER_PASSWORD2
                    def newUserEmail = env.NEW_USER_EMAIL
                    def roleName = env.ROLE_NAME

                    // Check if the new username already exists
                    def existingUsersResponse
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        existingUsersResponse = sh(
                            returnStdout: true,
                            script: "curl -s -X GET -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}asynchPeople/api/json?pretty=true'"
                        )
                    }
                    
                    // Parse JSON using jq
                    def existingUsersList = sh(
                        returnStdout: true,
                        script: "echo '${existingUsersResponse}' | jq -r '.users[].user.fullName'"
                    ).trim().split("\\s+")
                    
                    while (existingUsersList.contains(newUserName)) {
                        echo "User '${newUserName}' already exists."
                        newUserName = input(
                            id: 'userInput',
                            message: 'Please enter a new username:',
                            parameters: [
                                string(defaultValue: "", description: 'New username', name: 'NEW_USER')
                            ]
                        )
                        if (!newUserName) {
                            error("No new username provided. Pipeline aborted.")
                        }
        
                        // Update the existing users list
                        existingUsersList = sh(
                            returnStdout: true,
                            script: "echo '${existingUsersResponse}' | jq -r '.users[].user.fullName'"
                        ).trim().split("\\s+")
                    }
                    
                    if (newUserPassword1 != newUserPassword2) {
                        error("Passwords do not match. Please make sure both passwords are the same.")
                    } else {
                        echo "${newUserName}"
                        // Create the new user using the provided curl command
                        def createUserCmd = "curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' -d 'username=${newUserName}&password1=${newUserPassword1}&password2=${newUserPassword2}&fullname=${newUserName}&email=${newUserEmail}&type=hudson.security.HudsonPrivateSecurityRealm'"
                        def createUserResponse = withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                            sh(
                                returnStdout: true,
                                script: "${createUserCmd} -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}securityRealm/createAccount'"
                            )
                        }
                        echo "Response from Create User API: ${createUserResponse.trim()}"
                    }   
                }
                echo "Finished the 'Create Jenkins User' stage."
            }
        }

        stage('Add User to Role') {
            steps {
                script {
                    // Fetch existing roles using a GET request to the Role-Based Strategy API
                    def existingRolesResponse
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        existingRolesResponse = sh(
                            returnStdout: true,
                            script: "curl -s -X GET -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}role-strategy/strategy/getAllRoles'"
                        )
                    }
                    def existingRoleNames = sh(
                        returnStdout: true,
                        script: "echo '${existingRolesResponse}' | jq -r 'keys[]'"
                    ).trim().split("\\s+")
        
                    // Check if the new username already exists
                    def existingUsersResponse
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        existingUsersResponse = sh(
                            returnStdout: true,
                            script: "curl -s -X GET -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}asynchPeople/api/json?pretty=true'"
                        )
                    }
                    def existingUserNames = sh(
                        returnStdout: true,
                        script: "echo '${existingUsersResponse}' | jq -r '.users[].user.fullName'"
                    ).trim().split("\\n")
        
                   // Create a list of ChoiceParameterDefinition options for newUser and roleName
                    def newUserChoices = existingUserNames.collect { userName -> userName }
                    def roleNameChoices = existingRoleNames.collect { roleName -> roleName }
        
                    // Ask for input to specify the new user and role
                    def newUser = input(
                        id: 'newUserInput',
                        message: 'Please enter the new user:',
                        parameters: [
                            [$class: 'ChoiceParameterDefinition', choices: newUserChoices, description: "Existing Users: ${existingUserNames.join(', ')}", name: 'NEW_USER']
                        ]
                    )
        
                    def roleName = input(
                        id: 'roleInput',
                        message: 'Please select the role to add the user to:',
                        parameters: [
                            [$class: 'ChoiceParameterDefinition', choices: roleNameChoices, description: "Existing Roles: ${existingRoleNames.join(', ')}", name: 'ROLE_NAME']
                        ]
                    )
        
        
                    // Add the new user to the specified role
                    def addToRoleCmd = "curl -X POST -H 'Content-Type: application/x-www-form-urlencoded' -d 'type=globalRoles&roleName=${roleName}&sid=${newUser}'"
                    def addToRoleResponse
                    try {
                        addToRoleResponse = withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                            sh(
                                returnStdout: true,
                                script: "${addToRoleCmd} -u '${JENKINS_USER}:${JENKINS_API_TOKEN}' '${JENKINS_URL}role-strategy/strategy/assignRole'"
                            )
                        }
                    } catch (Exception ex) {
                        error("Failed to add user to role. Error: ${ex.message}")
                    }
        
                    echo "Response from Add User to Role API: ${addToRoleResponse.trim()}"
                }
                echo "Finished the 'Add User to Role' stage."
            }
        }
    }
}
