// pipeline {
//     agent any

//     stages {
//         stage('Validate Base Image') {
//             steps {
//                 script {
//                     // Checkout the repo
//                     git url: 'https://github.com/Sohailmd623/testing-governance.git', branch: 'main'

//                     // Approved images and versions
//                     def approvedImages = [
//                         "java/openjdk": ["8", "11", "17", "21"],
//                         "nodejs": ["20", "22", "24"],
//                         "dotnet": ["8", "9"],
//                         "python": ["3.9", "3.10", "3.11", "3.12"],
//                         "nginx": ["1.27", "1.28", "1.29"]
//                     ]

//                     // Read Dockerfile
//                     def dockerfileContent = readFile('Dockerfile')

//                     // Extract the FROM line
//                     def fromLine = dockerfileContent.readLines().find { it.trim().toUpperCase().startsWith("FROM") }

//                     if (!fromLine) {
//                         error("No FROM line found in Dockerfile!")
//                     }

//                     echo "Found base image: ${fromLine}"

//                     // Parse full image name and tag
//                     def matcher = fromLine =~ /FROM\s+([^\s:]+):?([^\s]*)/
//                     if (!matcher) {
//                         error("Unable to parse base image and tag from FROM line")
//                     }

//                     def fullImageName = matcher[0][1]  // e.g., drnzy.com/approved_images/node/nodejs-21
//                     def imageTag      = matcher[0][2]  // e.g., 10

//                     // Normalize image name for validation
//                     def imageName
//                     if (fullImageName ==~ /.*java\/openjdk$/) {
//                         imageName = "java/openjdk"
//                     } else if (fullImageName ==~ /.*node\/nodejs-\d+$/) {
//                         imageName = "nodejs"
//                     } else if (fullImageName ==~ /.*dotnet$/) {
//                         imageName = "dotnet"
//                     } else if (fullImageName ==~ /.*python$/) {
//                         imageName = "python"
//                     } else if (fullImageName ==~ /.*nginx$/) {
//                         imageName = "nginx"
//                     } else {
//                         error("Image '${fullImageName}' is not in the approved list!")
//                     }

//                     // Validate against approved images
//                     if (!approvedImages.containsKey(imageName)) {
//                         error("Image '${fullImageName}' is not in the approved list!")
//                     }

//                     if (!approvedImages[imageName].contains(imageTag)) {
//                         error("Tag '${imageTag}' for image '${fullImageName}' is not approved!")
//                     }

//                     echo "Base image '${fullImageName}:${imageTag}' is approved."
//                 }
//             }
//         }
//     }
// }


pipeline {
    agent any
    
    parameters {
        choice(
            name: 'GOVERNANCE_ACTION',
            choices: ['WARN_ONLY', 'FAIL_ON_EOL', 'FAIL_AFTER_GRACE_PERIOD'],
            description: 'Choose governance action for EOL images'
        )
        string(
            name: 'GRACE_PERIOD_MONTHS',
            defaultValue: '6',
            description: 'Grace period in months after EOL before failing builds'
        )
        string(
            name: 'WARNING_PERIOD_MONTHS',
            defaultValue: '6',
            description: 'Months before EOL to start warning'
        )
    }
    
    environment {
        ENDOFLIFE_API_BASE = 'https://endoflife.date/api/v1'
        APPROVED_IMAGE_REGISTRY = 'jfog.com/xyz_approved_images0'
        DOCKERFILE_PATH = 'Dockerfile'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Checkout the repository
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '*/master']],
                        userRemoteConfigs: [[url: 'https://github.com/your-org/your-repo.git']]
                    ])
                }
            }
        }
        
        stage('Parse Dockerfile') {
            steps {
                script {
                    echo "=== Parsing Dockerfile for base images ==="
                    
                    if (!fileExists(env.DOCKERFILE_PATH)) {
                        error "Dockerfile not found at ${env.DOCKERFILE_PATH}"
                    }
                    
                    def dockerfileContent = readFile(env.DOCKERFILE_PATH)
                    def fromLines = []
                    
                    // Extract all FROM lines (for multi-stage builds)
                    dockerfileContent.eachLine { line ->
                        if (line.trim().toUpperCase().startsWith('FROM')) {
                            fromLines.add(line.trim())
                        }
                    }
                    
                    if (fromLines.isEmpty()) {
                        error "No FROM statements found in Dockerfile"
                    }
                    
                    env.FROM_LINES = fromLines.join('|')
                    echo "Found FROM lines: ${fromLines}"
                }
            }
        }
        
        stage('Extract Base Images') {
            steps {
                script {
                    echo "=== Extracting and validating base images ==="
                    
                    def fromLines = env.FROM_LINES.split('\\|')
                    def baseImages = []
                    def approvedImages = getSupportedImages()
                    
                    fromLines.each { line ->
                        // Skip scratch and build stage references
                        if (line.contains('scratch') || line.contains(' AS ')) {
                            def parts = line.split(' AS ')
                            if (parts.length > 1) {
                                line = parts[0].trim()
                            }
                        }
                        
                        // Extract image name from FROM line
                        def imageParts = line.replaceAll(/^FROM\s+/, '').trim()
                        
                        if (!imageParts.contains('scratch')) {
                            def imageInfo = parseImageString(imageParts)
                            if (imageInfo) {
                                baseImages.add(imageInfo)
                            }
                        }
                    }
                    
                    if (baseImages.isEmpty()) {
                        error "No valid base images found to check"
                    }
                    
                    // Store base images for next stage
                    env.BASE_IMAGES_JSON = groovy.json.JsonOutput.toJson(baseImages)
                    echo "Extracted base images: ${baseImages}"
                }
            }
        }
        
        stage('Check Image Approval') {
            steps {
                script {
                    echo "=== Checking if images are from approved registry ==="
                    
                    def baseImages = new groovy.json.JsonSlurper().parseText(env.BASE_IMAGES_JSON)
                    def unapprovedImages = []
                    
                    baseImages.each { image ->
                        if (!image.fullImage.startsWith(env.APPROVED_IMAGE_REGISTRY)) {
                            unapprovedImages.add(image.fullImage)
                        }
                    }
                    
                    if (!unapprovedImages.isEmpty()) {
                        error """
                        GOVERNANCE VIOLATION: Unapproved base images detected!
                        
                        Unapproved images:
                        ${unapprovedImages.join('\n')}
                        
                        Please use images from approved registry: ${env.APPROVED_IMAGE_REGISTRY}
                        
                        Supported technologies and versions:
                        ${getSupportedImagesText()}
                        """
                    }
                    
                    echo "✅ All base images are from approved registry"
                }
            }
        }
        
        stage('Check EOL Status') {
            steps {
                script {
                    echo "=== Checking End-of-Life status for base images ==="
                    
                    def baseImages = new groovy.json.JsonSlurper().parseText(env.BASE_IMAGES_JSON)
                    def eolResults = []
                    def warnings = []
                    def errors = []
                    
                    baseImages.each { image ->
                        echo "Checking EOL status for: ${image.technology}:${image.version}"
                        
                        try {
                            def eolData = checkEOLStatus(image.technology, image.version)
                            
                            if (eolData) {
                                def result = [
                                    image: image,
                                    eolData: eolData,
                                    status: determineImageStatus(eolData, params.WARNING_PERIOD_MONTHS as Integer, params.GRACE_PERIOD_MONTHS as Integer)
                                ]
                                
                                eolResults.add(result)
                                
                                // Handle different statuses
                                switch(result.status.level) {
                                    case 'WARNING':
                                        warnings.add("⚠️  ${image.fullImage}: ${result.status.message}")
                                        break
                                    case 'ERROR':
                                        errors.add("❌ ${image.fullImage}: ${result.status.message}")
                                        break
                                    case 'OK':
                                        echo "✅ ${image.fullImage}: ${result.status.message}"
                                        break
                                }
                            }
                        } catch (Exception e) {
                            echo "⚠️  Could not check EOL status for ${image.technology}:${image.version} - ${e.getMessage()}"
                        }
                    }
                    
                    // Display warnings
                    if (!warnings.isEmpty()) {
                        echo """
                        === WARNINGS ===
                        ${warnings.join('\n')}
                        """
                    }
                    
                    // Handle errors based on governance action
                    if (!errors.isEmpty()) {
                        def errorMessage = """
                        === EOL VIOLATIONS DETECTED ===
                        ${errors.join('\n')}
                        
                        Supported versions:
                        ${getSupportedImagesText()}
                        
                        Please update to supported versions to continue.
                        """
                        
                        switch(params.GOVERNANCE_ACTION) {
                            case 'WARN_ONLY':
                                echo "🚨 ${errorMessage}"
                                echo "⚠️  Build continuing due to WARN_ONLY mode"
                                break
                            case 'FAIL_ON_EOL':
                                error errorMessage
                                break
                            case 'FAIL_AFTER_GRACE_PERIOD':
                                error errorMessage
                                break
                        }
                    }
                    
                    echo "=== EOL Status Check Complete ==="
                }
            }
        }
        
        stage('Generate EOL Report') {
            steps {
                script {
                    echo "=== Generating EOL Status Report ==="
                    
                    def reportContent = generateEOLReport()
                    
                    // Write report to file
                    writeFile file: 'eol-report.json', text: reportContent
                    
                    // Archive the report
                    archiveArtifacts artifacts: 'eol-report.json', fingerprint: true
                    
                    echo "📊 EOL report generated and archived"
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Clean up temporary files if any
                sh 'rm -f temp-eol-*.json || true'
            }
        }
        success {
            echo "✅ Base image governance check passed successfully"
        }
        failure {
            echo "❌ Base image governance check failed"
        }
    }
}

// Helper Functions
def parseImageString(imageString) {
    try {
        // Handle registry/namespace/image:tag format
        def parts = imageString.split('/')
        def imageAndTag = parts[-1]
        
        def registry = ""
        def namespace = ""
        
        if (parts.length >= 3) {
            registry = parts[0]
            namespace = parts[1..-2].join('/')
        } else if (parts.length == 2) {
            namespace = parts[0]
        }
        
        def imageParts = imageAndTag.split(':')
        def imageName = imageParts[0]
        def tag = imageParts.length > 1 ? imageParts[1] : 'latest'
        
        // Map image names to endoflife.date product names
        def technologyMap = [
            'java': 'java',
            'openjdk': 'java',
            'node': 'nodejs',
            'nodejs': 'nodejs',
            'dotnet': 'dotnet',
            'python': 'python',
            'nginx': 'nginx'
        ]
        
        def technology = technologyMap[imageName.toLowerCase()]
        
        if (!technology) {
            echo "⚠️  Unknown technology: ${imageName}"
            return null
        }
        
        // Extract version number from tag
        def version = extractVersionFromTag(tag, technology)
        
        return [
            fullImage: imageString,
            registry: registry,
            namespace: namespace,
            imageName: imageName,
            tag: tag,
            technology: technology,
            version: version
        ]
    } catch (Exception e) {
        echo "Error parsing image string: ${imageString} - ${e.getMessage()}"
        return null
    }
}

def extractVersionFromTag(tag, technology) {
    // Handle common tag patterns
    switch(technology) {
        case 'java':
            // Handle openjdk:8, openjdk:11, java:17-jre, etc.
            def matcher = tag =~ /(\d+)/
            return matcher ? matcher[0][1] : tag
        case 'nodejs':
            // Handle node:18, node:18.20.8, node:18-alpine, etc.
            def matcher = tag =~ /(\d+)/
            return matcher ? matcher[0][1] : tag
        case 'dotnet':
            // Handle dotnet:8.0, dotnet:9.0-sdk, etc.
            def matcher = tag =~ /(\d+)/
            return matcher ? matcher[0][1] : tag
        case 'python':
            // Handle python:3.11, python:3.11-slim, etc.
            def matcher = tag =~ /(\d+\.\d+)/
            return matcher ? matcher[0][1] : tag
        case 'nginx':
            // Handle nginx:1.27, nginx:1.27-alpine, etc.
            def matcher = tag =~ /(\d+\.\d+)/
            return matcher ? matcher[0][1] : tag
        default:
            return tag
    }
}

def checkEOLStatus(technology, version) {
    try {
        def url = "${env.ENDOFLIFE_API_BASE}/products/${technology}/releases/${version}"
        echo "Checking EOL API: ${url}"
        
        def response = sh(
            script: "curl -s -X GET '${url}' -H 'accept: application/json'",
            returnStdout: true
        ).trim()
        
        if (response.startsWith('{')) {
            def jsonResponse = new groovy.json.JsonSlurper().parseText(response)
            return jsonResponse.result
        } else {
            echo "Invalid response from EOL API: ${response}"
            return null
        }
    } catch (Exception e) {
        echo "Error checking EOL status: ${e.getMessage()}"
        return null
    }
}

def determineImageStatus(eolData, warningMonths, graceMonths) {
    def currentDate = new Date()
    
    if (!eolData.eolFrom) {
        return [level: 'OK', message: 'No EOL date available - assumed supported']
    }
    
    def eolDate = Date.parse("yyyy-MM-dd", eolData.eolFrom.split('T')[0])
    def daysDifference = (eolDate.time - currentDate.time) / (1000 * 60 * 60 * 24)
    
    if (daysDifference > (warningMonths * 30)) {
        return [level: 'OK', message: "Supported until ${eolData.eolFrom}"]
    } else if (daysDifference > 0) {
        def daysUntilEol = Math.ceil(daysDifference)
        return [level: 'WARNING', message: "Will reach EOL in ${daysUntilEol} days (${eolData.eolFrom}). Please plan to upgrade."]
    } else {
        def daysSinceEol = Math.ceil(Math.abs(daysDifference))
        def graceExpired = daysSinceEol > (graceMonths * 30)
        
        if (graceExpired || params.GOVERNANCE_ACTION == 'FAIL_ON_EOL') {
            return [level: 'ERROR', message: "EOL since ${eolData.eolFrom} (${daysSinceEol} days ago). Must upgrade to continue."]
        } else {
            return [level: 'WARNING', message: "EOL since ${eolData.eolFrom} (${daysSinceEol} days ago). Grace period active."]
        }
    }
}

def getSupportedImages() {
    return [
        'java': ['8', '11', '17', '21'],
        'nodejs': ['20', '22', '24'],
        'dotnet': ['8', '9'],
        'python': ['3.9', '3.10', '3.11', '3.12'],
        'nginx': ['1.27', '1.28', '1.29']
    ]
}

def getSupportedImagesText() {
    def supported = getSupportedImages()
    def text = []
    
    supported.each { tech, versions ->
        text.add("${tech}: ${versions.join(', ')}")
    }
    
    return text.join('\n')
}

def generateEOLReport() {
    def report = [
        timestamp: new Date().format("yyyy-MM-dd'T'HH:mm:ss'Z'"),
        repository: env.GIT_URL ?: 'unknown',
        branch: env.GIT_BRANCH ?: 'unknown',
        buildNumber: env.BUILD_NUMBER ?: 'unknown',
        governanceAction: params.GOVERNANCE_ACTION,
        warningPeriodMonths: params.WARNING_PERIOD_MONTHS,
        gracePeriodMonths: params.GRACE_PERIOD_MONTHS,
        images: []
    ]
    
    try {
        def baseImages = new groovy.json.JsonSlurper().parseText(env.BASE_IMAGES_JSON ?: '[]')
        report.images = baseImages
    } catch (Exception e) {
        echo "Could not include image details in report: ${e.getMessage()}"
    }
    
    return groovy.json.JsonOutput.prettyPrint(groovy.json.JsonOutput.toJson(report))
}
