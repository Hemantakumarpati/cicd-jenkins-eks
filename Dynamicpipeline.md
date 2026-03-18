# Dynamic Pipeline Generation in Jenkins

This guide explains the scenarios where dynamic pipelines are highly effective and provides a step-by-step tutorial on building one utilizing your existing Jenkins Shared Library setup.

---

## 1. Scenarios: When to Use a Dynamic Pipeline

Dynamic pipelines shine when automation needs to be flexible, scalable, and entirely data-driven. You should use a dynamic pipeline in the following scenarios:

### Scenario A: Monorepos with Multiple Microservices
If you manage a single repository holding 15 microservices, writing a standard `Jenkinsfile` would require manually defining 15 `stage('Build X')` blocks. If a developer adds a 16th microservice, they must submit a PR to modify the Jenkinsfile code. 
**Dynamic Solution:** The pipeline dynamically reads the repository's directories or a `services.json` file and auto-generates 15 parallel build stages on the fly.

### Scenario B: Matrix Testing Across Environments
You need to test an application across multiple Operating Systems (Linux, Windows, macOS) and Runtime versions (Java 11, 17, 21). 
**Dynamic Solution:** You loop over arrays of OS and Java versions to dynamically construct the test matrix, easily scaling up or down just by changing a variable or list.

### Scenario C: Configuration-Driven CI/CD
You want to hide complex Groovy CI/CD logic from developers. You only want them to place a simple `build-config.json` inside their repository, dictating what needs to be built and deployed without them ever touching the Groovy code.
**Dynamic Solution:** Your Shared Library pipeline fetches this JSON file, parses the parameters, and builds the Jenkins stages dynamically based on what the developer requested in the file.

---

## 2. Step-by-Step Guide: Implementing a Dynamic Pipeline

Since you have already configured a Jenkins Shared Library, the best approach is to store the "engine" (the Groovy logic that constructs the stages) in the Shared Library. The application repository will just supply the data.

### Step 1: Create a Configuration File in the Application Repository
In your application's source code repository (e.g., `cicd-java`), create a configuration file named `services.json`. This tells the pipeline what to build.

**File: `services.json` (in app repo)**
```json
{
  "microservices": [
    "user-service",
    "payment-service",
    "notification-service"
  ],
  "run_integration_tests": true
}
```

### Step 2: Write the Dynamic Logic in your Shared Library
Now, in your **Shared Library repository**, create a new custom step inside the `vars/` directory. We will call it `dynamicServicesBuilder.groovy`. This script will open the JSON configuration and dynamically generate parallel stages for everything it finds.

**File: `vars/dynamicServicesBuilder.groovy` (in Shared Library repo)**
```groovy
import groovy.json.JsonSlurperClassic

def call(Map config = [:]) {
    pipeline {
        agent any
        
        stages {
            stage('Initialize') {
                steps {
                    script {
                        echo "Reading configuration file: ${config.configFile}"
                    }
                }
            }
            
            stage('Build Microservices Dynamically') {
                steps {
                    script {
                        // 1. Read and parse the JSON configuration file from the app workspace
                        def configText = readFile(config.configFile)
                        def parsedConfig = new JsonSlurperClassic().parseText(configText)
                        
                        // 2. Create an empty map to hold our dynamically generated parallel stages
                        def parallelBuildStages = [:]
                        
                        // 3. Loop over the microservices defined in the JSON file
                        for (int i = 0; i < parsedConfig.microservices.size(); i++) {
                            def serviceName = parsedConfig.microservices[i]
                            
                            // 4. Dynamically create a stage construct for each service
                            parallelBuildStages[serviceName] = {
                                stage("Build ${serviceName}") {
                                    echo "Executing build tasks for ${serviceName}..."
                                    
                                    // Replace with your actual build command (e.g. Maven, Gradle, or Docker)
                                    // sh "mvn clean install -pl ${serviceName}"
                                    sh "echo 'Building ${serviceName}'"
                                }
                            }
                        }
                        
                        // 5. Execute all dynamically generated stages in parallel!
                        parallel parallelBuildStages
                    }
                }
            }
            
            stage('Integration Tests') {
                // Conditionally run a stage based on data from the JSON file
                when {
                    expression {
                        def configText = readFile(config.configFile)
                        def parsedConfig = new JsonSlurperClassic().parseText(configText)
                        return parsedConfig.run_integration_tests == true
                    }
                }
                steps {
                    echo "Running Integration Tests because it was enabled in the config!"
                }
            }
        }
    }
}
```

### Step 3: Call the Dynamic Builder in your Jenkinsfile
Finally, in your application repository, create or update your `Jenkinsfile`. Instead of containing hundreds of lines of redundant stages, it will simply invoke the step you created in your Shared Library and pass it the JSON file.

**File: `Jenkinsfile` (in the app repo)**
```groovy
// Import your configured shared library (replace 'my-shared-library' with your actual library name in Jenkins globally)
@Library('my-shared-library') _

// Call the dynamic builder and declare which config file to use
dynamicServicesBuilder(
    configFile: 'services.json'
)
```

### Summary of What Happens at Runtime:
1. Jenkins starts the build for your application repository using the `Jenkinsfile`.
2. It fetches your Shared Library and loads the `dynamicServicesBuilder` script.
3. The shared library step reads `services.json` from the application's workspace.
4. It parses the JSON, reads `user-service`, `payment-service`, and `notification-service`.
5. It uses a `for` loop to dynamically generate a parallel build map for each of those 3 services.
6. If a developer later pushes a commit that adds `"auth-service"` to `services.json`, **no Jenkinsfile or Groovy logic needs to change**. Jenkins will automatically detect the new entry and spin up a 4th parallel stage on the next build!
