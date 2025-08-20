# Implementation Plan

- [x] 1. Create missing checkstyle configuration and fix Maven setup


  - Create directory structure `src/misc/style/java/` and basic checkstyle.xml configuration file
  - Modify Maven pom.xml files to skip checkstyle validation temporarily with `-Dcheckstyle.skip=true`
  - Ensure Maven wrapper files have proper execute permissions in pipeline
  - _Requirements: 3.1, 3.3, 3.4_

- [x] 2. Implement build environment setup stage in Jenkins pipeline


  - Add "Build Environment Setup" stage after "Checkout Code" and before "Security Scans"
  - Add environment validation for Java 21, Maven, Go 1.23, and Node.js
  - Create stage to fix Maven wrapper permissions using `chmod +x ./src/*/mvnw`
  - Add validation checks for required build tools and versions
  - _Requirements: 3.1, 3.2, 3.4_


- [x] 3. Create Maven service build and test functionality


  - [ ] 3.1 Implement Maven build stage for cart service
    - Add "Build & Test Cart Service" stage with Maven clean compile and test execution
    - Use `./mvnw clean compile test -Dcheckstyle.skip=true` in src/cart directory
    - Create error handling for Maven build failures with detailed logging
    - Verify JAR artifact creation in target/ directory before proceeding


    - _Requirements: 1.1, 1.2, 2.1, 2.4_

  - [ ] 3.2 Implement Maven build stage for orders service
    - Add "Build & Test Orders Service" stage with Maven clean compile and test execution
    - Use `./mvnw clean compile test -Dcheckstyle.skip=true` in src/orders directory


    - Handle PostgreSQL and RabbitMQ dependencies for integration tests using Testcontainers
    - Verify JAR artifact creation and test results
    - _Requirements: 1.1, 1.2, 2.1, 2.4_

  - [ ] 3.3 Implement Maven build stage for ui service
    - Add "Build & Test UI Service" stage that runs after other services complete
    - Execute Kiota client generation first, then Maven clean compile and test


    - Use `./mvnw clean compile test -Dcheckstyle.skip=true` in src/ui directory
    - Handle dependency on other services' OpenAPI specs (../cart/openapi.yml, etc.)
    - _Requirements: 1.1, 1.2, 2.1, 2.4, 4.2_

- [ ] 4. Create Go service build and test functionality
  - Add "Build & Test Catalog Service" stage for Go service


  - Use `go mod download && go build ./...` and `go test ./...` in src/catalog directory
  - Implement Go module dependency resolution and caching with GOCACHE
  - Create error handling for Go compilation and test failures

  - Verify Go binary creation before Docker build
  - _Requirements: 1.1, 1.3, 2.2, 2.4_

- [ ] 5. Create Node.js service build and test functionality
  - Add "Build & Test Checkout Service" stage for Node.js service
  - Use `npm ci && npm run build` and `npm test` in src/checkout directory
  - Implement npm test execution with proper error handling (currently exits 0)


  - Handle NestJS compilation and dist folder creation
  - Verify build artifacts before proceeding to Docker build
  - _Requirements: 1.1, 1.4, 2.3, 2.4_

- [x] 6. Implement build orchestration and dependency management


  - Create parallel build execution for independent services (cart, orders, catalog, checkout)
  - Implement sequential build for UI service after other services complete (due to OpenAPI dependencies)
  - Add build status tracking and reporting for each service
  - Create build failure isolation to prevent one service failure from stopping others


  - _Requirements: 4.1, 4.2, 4.3, 5.4_

- [ ] 7. Replace current Docker-only stages with proper build-then-docker workflow
  - Remove existing "Build & Push UI Image", "Build & Push CART Image", etc. stages
  - Replace with new stages that first compile/build, then create Docker images



  - Ensure Docker builds use pre-compiled artifacts from build stages
  - Maintain existing Docker multi-stage build structure in Dockerfiles
  - _Requirements: 1.1, 1.5_

- [ ] 8. Implement comprehensive error handling and logging
  - Add detailed error logging for each build stage with service-specific context
  - Create build failure notifications with actionable error information
  - Implement build summary reporting showing success/failure status for each service
  - Add error recovery mechanisms for transient build failures
  - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

- [ ] 9. Add test result aggregation and reporting
  - Implement test result collection from Maven surefire reports (target/surefire-reports/)
  - Add Go test result parsing and reporting with `-json` output format
  - Create Node.js test result aggregation from Jest output
  - Generate unified test report across all services using Jenkins test reporting
  - _Requirements: 2.1, 2.2, 2.3, 2.4_

- [ ] 10. Optimize pipeline performance and caching
  - Implement Maven local repository caching between builds using Jenkins workspace
  - Add Go module cache persistence for faster dependency resolution with GOCACHE
  - Create Node.js node_modules caching strategy using npm ci
  - Add Docker layer caching for improved build performance
  - _Requirements: 3.2, 4.3_