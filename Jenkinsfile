def findApiNameinFile(fileName) {
    def props = readYaml file: fileName
    return props.info.title
}
def findPathsinFile(fileName) {
    def props = readYaml file: fileName 
    return props.paths
}
def findPoliciesInFile(fileName,path,method) {
    def props = readYaml file: fileName 
    println 'founded paths: ' + props.paths.getAt(path)//.find{pth->pth == path}
	return props.paths.getAt(path).getAt(method).policies
}
def findApiIdByName(name, json) {
    def result = ""
    json.items.eachWithIndex { val, idx ->
        if ( val.name == name ) {
            println " $name found with id "+val.id
            result = val.id
        }
    }
    return result
}
def findEndpointsInPath(path) {
    def returnMap = [:]
    path.getValue().keySet().each {method-> returnMap.put(method, path.getKey())}
    return returnMap
}
def findEndpointsInPath2(path) {
    def returnMap = [:]
    // println "Path2 - path.getKey(): " + path.getKey()//.keySet().each {method-> returnMap.put(method, path.getKey())}
    returnMap.put( path.getKey(), new ArrayList())
    path.getValue().keySet().each{method -> returnMap.get(path.getKey()).add(method)}
    // println "Path2 - returnMap.get(): " + returnMap.get(path.getKey())//.add(path.getValues().keySet())
    return returnMap
}
def intersectPaths(swaggerPaths, integrationPaths) {
    println "swaggerPaths: " + swaggerPaths
    println "integrationPaths: " + integrationPaths
    def returnMap = [:]
    swaggerPaths.each {
        swagger ->  integrationPaths.each { integr -> 
            if(integr.key == swagger.key) {
                returnMap.put(swagger.key, integr.value.intersect(swagger.value))
            }
        }
    }
    return returnMap
}
def combinePaths(paths, swaggerFile, integrationFile) {
    def returnMap = [:]
    def swaggerProps = readYaml file: swaggerFile 
    def integrationProps = readYaml file: integrationFile 
    swaggerProps.keySet()
        .each { key -> 
            copyAttributes:{
                if (key == "paths") {
                    def patt = new LinkedHashMap()
                    paths.each {
                        path -> L:{
                            def methods = new LinkedHashMap()
                            path.value.each {
                                method -> L:{
                                    def methodBody = swaggerProps.get(key).get(path.key).get(method)
                                    methodBody.put("x-amazon-apigateway-integration",integrationProps.get(key).get(path.key).get(method).get("x-amazon-apigateway-integration"))
                                    methods.put(method, methodBody)
                                }
                            }
                            patt.put(path.key, methods)
                        }
                    }
                    returnMap.put(key, patt)
                }
                else 
                    returnMap.put(key, swaggerProps.get(key))
            }
        }
    return returnMap;
}
def getActualPolicyAuthorizerConfiguration() {
	def appConfigApplications = readYaml text: sh (label: 'Retrieve list applications', script: "aws appconfig list-applications", returnStdout: true)
	def policyAuthorizerId = appConfigApplications.Items.find{it.Name == 'policyAuthorizer'}.Id
	env.policyAuthorizerId = policyAuthorizerId
	def applicationProfiles =  readYaml text: sh (label: 'Retrieve configuration profiles', script: "aws appconfig list-configuration-profiles --application-id ${policyAuthorizerId}", returnStdout: true)
	def profileId = applicationProfiles.Items.find{it.Name == 'endpoint-public'}.Id
	env.profileId = profileId
	def lastDeployment = readYaml text: sh (label: 'Retrieve configuration profile', script: "aws appconfig list-deployments --application-id ${policyAuthorizerId} --environment-id 3ynyst4 --max-results 1", returnStdout: true)
	env.lastDeployment = lastDeployment
	def actualConfigurationVersion = lastDeployment.Items.find{it.State == 'COMPLETE'}.ConfigurationVersion
	
	sh (label: 'Start configuration session', script: "aws appconfig get-hosted-configuration-version --application-id ${policyAuthorizerId} --configuration-profile-id ${profileId} --version-number ${actualConfigurationVersion} policyAuthorizerConfiguration.yaml", returnStdout: true)			
	
}

def getConfigurtion() {
    getActualPolicyAuthorizerConfiguration()
	def conf = readYaml file:"policyAuthorizerConfiguration.yaml"
	return conf
}


def swaggerPaths = []
def integrationPaths = []
def String yaml
def actualPolicyAuthorizerConfiguration
def apiConfigurationActions = []
def newApisConfigurations = []

pipeline {
    agent {label "built-in" }
    environment {
        AWS_DEFAULT_REGION="eu-west-1"
    }
    stages {
        stage('checkin API swaggerPaths') {
            steps {
                git branch: 'main', url: 'https://github.com/Azbest78/JenkinsTest'
                script {
                    env.filename_api_var = sh(label: 'Retrieve changed file',  script: 'git diff --name-only HEAD^ HEAD', returnStdout: true ).trim()
                    //  env.filename_api_var.endsWith("-swagger.yaml")
                    println env.filename_api_var
                    env.api_var = findApiNameinFile("public/dev-TestMicroServiceAPI-swagger.yaml")
                    def paths = findPathsinFile("public/dev-TestMicroServiceAPI-swagger.yaml")
                    paths.each {path -> findEndpointsInPath2(path).each {endpoint -> swaggerPaths.add(endpoint)}}
                    swaggerPaths.each {endpoint -> println endpoint.value + ' ' + endpoint.key}
                }
            }
        }
        stage('checkin API integrationPaths') {
            steps {
                 script {
                     // env.filename_api_var = sh(label: 'Retrieve changed file',  script: 'git diff --name-only HEAD^ HEAD', returnStdout: true ).trim()
                     // env.api_var = findApiNameinFile(env.filename_api_var)
                     def paths = findPathsinFile("integration/dev-TestMicroServiceAPI-integration.yaml")
                     paths.each {path -> findEndpointsInPath2(path).each {endpoint -> integrationPaths.add(endpoint)}}
                     integrationPaths.each {endpoint -> println endpoint.value + ' ' + endpoint.key}
                 }
            }
        }
        stage('union API swaggerPaths and integrationPaths') {
            steps {
                script {
                    def paths = intersectPaths(swaggerPaths, integrationPaths)
                    def combinedPath = combinePaths(paths, "public/dev-TestMicroServiceAPI-swagger.yaml", "integration/dev-TestMicroServiceAPI-integration.yaml")
                    yaml = writeYaml returnText: true, data: combinedPath
                }
            }
        }
        stage('import-rest-api to AWS') {
			steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: "aws-jenkins", secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' )]){
                    script {
                        response = sh(label: 'Retrieve api Info', script: "aws apigateway get-rest-apis", returnStdout: true)
                        def json = readYaml text: response
                        env.id = findApiIdByName(env.api_var, json)
                        if (id == '') {
                             echo "findApiIdByName is empty. creating new api "
                             importRespone = sh (label: 'import new api', script: "aws apigateway import-rest-api --body '${yaml}'", returnStdout: true)
                             def importResponseJson = readJSON text: importRespone
                             env.id = importResponseJson.id
                         } else {
                             println "update an api"
                             sh("aws apigateway put-rest-api --rest-api-id ${env.id} --mode overwrite --body '${yaml}'")
                         }
                    }
                }
            }
        }
        stage('deploy api to DEV stage') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: "aws-jenkins", secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' )]){
                    sh ("aws apigateway create-deployment --rest-api-id ${env.id} --stage-name dev --description 'deployment build ${env.BUILD_ID} to the dev stage'")
                }
			}
        }
        stage('get info about policyAuthorizer application') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: "aws-jenkins", secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' )])
				{
					script {
						def newApi = [:]
						def isNneedToBeUpdated = false;
						actualPolicyAuthorizerConfiguration = getConfigurtion()
						def apiConfiguration = actualPolicyAuthorizerConfiguration.api.find{it.name == env.api_var}
						if (apiConfiguration == null) isNneedToBeUpdated = true;
						println 'apiConfiguration' + apiConfiguration
						println 'Configuration for api ${env.api_var.value} exist: ' + apiConfiguration != null
						println 'test3'
						newApi.put('name', env.api_var)
						
						//apiConfigurationActions.put(env.api_var, apiConfiguration == null)
						def configuredEndpoints = apiConfiguration.endpoints
						def endpointActions = []
						intersectPaths(swaggerPaths, integrationPaths).each{intersectPath -> L:{
							
							def confEndpoint = configuredEndpoints.find{endpoint -> endpoint.path == intersectPath.key}
							echo 'confEndpoint: ' + confEndpoint
							def endpointAction = [:]
							endpointAction.put('path', intersectPath.key)
                            if(apiConfiguration == null) isNneedToBeUpdated = true;
				// 			endpointAction.put('toAdd', confEndpoint == null)
							
							def methodsActions =[]
							
							intersectPath.value.each{ pathValue -> O:{
								def methodAction = [:]
								def confMethod
								def policiesInIntegration = findPoliciesInFile('integration/dev-TestMicroServiceAPI-integration.yaml', intersectPath.key, pathValue).sort()
									
								if(confEndpoint != null) {
								    confMethod = confEndpoint.methods.find{method -> method.name == pathValue.toUpperCase()}
								} 
								methodAction.put('name', pathValue.toUpperCase())	
								if(confMethod == null) isNneedToBeUpdated = true;
								// methodAction.put('to_add', confMethod == null)
								methodAction.put('policies', policiesInIntegration)
								if(confMethod == null)
									isNneedToBeUpdated = true;
								else {
								    // methodAction.put('policies_update', !policiesInIntegration.equals(confMethod.policies.sort()))
								    if (!policiesInIntegration.equals(confMethod.policies.sort())) isNneedToBeUpdated = true
								}

								methodsActions.add(methodAction)
							}}
							endpointAction.put('methods', methodsActions)
						    endpointActions.add(endpointAction)	
						}}
						newApi.put('endpoints', endpointActions)
						if(isNneedToBeUpdated) apiConfigurationActions.add(env.api_var)
						newApisConfigurations.add(newApi)


					}
				}
			}
        }
        stage('policyAuthorizer configuration release') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: "aws-jenkins", secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' )])
				{
				    script {
				        if(apiConfigurationActions.size() > 0) {
				            println 'apiConfigurationActions size: ' + apiConfigurationActions.size()
				         
				      
				            def newConf = ['version': actualPolicyAuthorizerConfiguration.version]
				            def apis = []
				            actualPolicyAuthorizerConfiguration.api.each{api->OL:{
				                if(!apiConfigurationActions.contains(api.name)) 
				                    apis.add(api)
				            }}
				            
				            newApisConfigurations.each{api-> apis.add(api)}
				            
				            newConf.put('api', apis)
				            println 'newConf: ' + newConf
				            def returnJson = writeJSON returnText: true, json: newConf
				            
				            writeJSON file: 'newConf.json', json: newConf
							def base64content = returnJson.bytes.encodeBase64().toString()
				            sh ("aws appconfig create-hosted-configuration-version --application-id ${env.policyAuthorizerId} --configuration-profile-id ${env.profileId} --content-type 'application/json' --content '${returnJson}' result.json ")

				        }
                    }
				}
            }
        }
		stage('policyAuthorizer configuration deploy') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: "aws-jenkins", secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' )])
				{
				    script {
				        if(apiConfigurationActions.size() > 0) {
							def lastConfig = readJSON text: sh (label: 'Retrieve list of configurations', script: "aws appconfig list-hosted-configuration-versions --application-id ${env.policyAuthorizerId} --configuration-profile-id ${env.profileId}", returnStdout: true)
				            sh ("aws appconfig start-deployment --application-id ${env.policyAuthorizerId} --environment-id 3ynyst4 --deployment-strategy-id l8kz56f --configuration-profile-id ${env.profileId} --configuration-version ${lastConfig.Items.get(0).VersionNumber}")

				        }
                    }
				}
            }
        }
    }
}