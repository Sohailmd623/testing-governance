pipeline {
    agent any

    stages {
        stage('Validate Base Image') {
            steps {
                script {
                    // Checkout the repo
                    git url: 'https://github.com/Sohailmd623/testing-governance.git', branch: 'main'

                    // Approved images and versions
                    def approvedImages = [
                        "java/openjdk": ["8", "11", "17", "21"],
                        "nodejs": ["20", "22", "24"],
                        "dotnet": ["8", "9"],
                        "python": ["3.9", "3.10", "3.11", "3.12"],
                        "nginx": ["1.27", "1.28", "1.29"]
                    ]

                    // Read Dockerfile
                    def dockerfileContent = readFile('Dockerfile')

                    // Extract the FROM line
                    def fromLine = dockerfileContent.readLines().find { it.trim().toUpperCase().startsWith("FROM") }

                    if (!fromLine) {
                        error("No FROM line found in Dockerfile!")
                    }

                    echo "Found base image: ${fromLine}"

                    // Parse full image name and tag
                    def matcher = fromLine =~ /FROM\s+([^\s:]+):?([^\s]*)/
                    if (!matcher) {
                        error("Unable to parse base image and tag from FROM line")
                    }

                    def fullImageName = matcher[0][1]  // e.g., drnzy.com/approved_images/node/nodejs-21
                    def imageTag      = matcher[0][2]  // e.g., 10

                    // Normalize image name for validation
                    def imageName
                    if (fullImageName ==~ /.*java\/openjdk$/) {
                        imageName = "java/openjdk"
                    } else if (fullImageName ==~ /.*node\/nodejs-\d+$/) {
                        imageName = "nodejs"
                    } else if (fullImageName ==~ /.*dotnet$/) {
                        imageName = "dotnet"
                    } else if (fullImageName ==~ /.*python$/) {
                        imageName = "python"
                    } else if (fullImageName ==~ /.*nginx$/) {
                        imageName = "nginx"
                    } else {
                        error("Image '${fullImageName}' is not in the approved list!")
                    }

                    // Validate against approved images
                    if (!approvedImages.containsKey(imageName)) {
                        error("Image '${fullImageName}' is not in the approved list!")
                    }

                    if (!approvedImages[imageName].contains(imageTag)) {
                        error("Tag '${imageTag}' for image '${fullImageName}' is not approved!")
                    }

                    echo "Base image '${fullImageName}:${imageTag}' is approved."
                }
            }
        }
    }
}
