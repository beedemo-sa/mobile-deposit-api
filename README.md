# Beedemo Mobile Deposit API
-----------------------------
A simple Docker Spring Boot Jersey 2 API for use by the Beedemo mobile-deposit-ui example. Uses a Jenkinsfile with the declarative syntax to automatically create a Jenkins Pipeline job.

###Features
- Uses the `branch_name` variable to control stages.
- Uses Pipeline Global Library to build and push and Docker image and to deploy the Docker image.
