# Drupal Module Testing with GitHub Actions

This README provides a comprehensive guide for setting up automated testing of Drupal modules using GitHub Actions.

## Overview

This setup allows you to run PHPUnit tests for your Drupal custom module in a GitHub Actions workflow. The workflow:

1. Sets up a PHP environment
2. Installs Drupal with a MySQL database
3. Enables your custom module and its dependencies
4. Starts a web server
5. Runs your PHPUnit tests

## Requirements

### For Local Development and Testing

* PHP 8.0+ with required extensions (mbstring, pdo, xml, dom, json)
* MySQL 5.7+
* Composer
* Drush
* [Docker Desktop](https://www.docker.com/products/docker-desktop/) with WSL2 backend (for Windows users)
* [act](https://github.com/nektos/act) for local GitHub Actions testing

### For GitHub Actions

* A GitHub repository with your Drupal module
* A properly configured `.github/workflows/phpunit.yml` file

## Local Setup Instructions

### Windows Setup

1. Install Docker Desktop with WSL2 backend:
   ```
   # Run PowerShell as Administrator
   choco install docker-desktop
   ```

2. Install act for local GitHub Actions testing:
   ```
   # Run PowerShell as Administrator
   choco install act-cli
   ```

3. Run Docker Desktop as Administrator

4. Always run act commands in an Administrator PowerShell:
   ```
   act -W .github/workflows/phpunit.yml -j build --container-daemon-socket "npipe:////./pipe/docker_engine"
   ```

### Linux/macOS Setup

1. Install Docker:
   ```
   # Ubuntu
   sudo apt-get update
   sudo apt-get install docker-ce docker-ce-cli containerd.io

   # macOS
   brew install --cask docker
   ```

2. Install act:
   ```
   # Ubuntu
   curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

   # macOS
   brew install act
   ```

3. Run act:
   ```
   act -W .github/workflows/phpunit.yml
   ```

## Configuration Files

These files should be placed in your project's root directory following standard Drupal structure:

- `.github/workflows/phpunit.yml` - GitHub Actions workflow file
- `phpunit.xml` - PHPUnit configuration
- `web/modules/custom/MODULE_NAME` - Your custom module with:
  - `tests/src/Unit/` - Unit tests
  - `tests/src/Kernel/` - Kernel tests
  - `tests/src/Functional/` - Functional tests
  - `tests/src/FunctionalJavascript/` - Browser tests

### GitHub Workflow File (.github/workflows/phpunit.yml)

```yaml
name: PHPUnit Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:5.7
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: MODULE_NAME_testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
      - uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo, xml, dom, json

      - name: Install Composer dependencies
        run: |
          composer install --no-interaction --no-progress

      - name: Install Drupal
        run: |
          composer require drush/drush
          ./vendor/bin/drush site:install standard -y \
            --db-url="mysql://root:root@mysql:3306/MODULE_NAME_testing" \
            --site-name="Test site" \
            --account-name=admin \
            --account-pass=admin

      - name: Enable required modules
        run: |
          cd web
          ../vendor/bin/drush pm:list --type=module | grep MODULE_NAME
          ../vendor/bin/drush en MODULE_NAME -y -v
          cd ..
          
          # Prepare testing environment
          mkdir -p /tmp/browser_output
          chmod -R 777 /tmp/browser_output web/sites/default/files web/sites/simpletest

      - name: Run PHPUnit tests
        run: |
          # Start PHP's built-in web server in the background
          cd web
          php -S 0.0.0.0:8080 .ht.router.php &
          SERVER_PID=$!
          echo "Started PHP server with PID: $SERVER_PID"
          
          # Wait for server to be ready
          sleep 3
          
          # Test connection
          curl -I http://127.0.0.1:8080 || echo "Web server not responding"
          
          # Run tests
          echo "Running tests:"
          SIMPLETEST_BASE_URL=http://127.0.0.1:8080 ../vendor/bin/phpunit --verbose --configuration ../phpunit.xml
          
          # Kill the PHP server
          kill $SERVER_PID || true
        env:
          WEBDRIVER_HOST: "selenium"
          SIMPLETEST_BASE_URL: "http://127.0.0.1:8080"
```

### PHPUnit Configuration File (phpunit.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         bootstrap="web/core/tests/bootstrap.php" colors="true"
         beStrictAboutTestsThatDoNotTestAnything="true"
         beStrictAboutOutputDuringTests="true"
         beStrictAboutChangesToGlobalState="true"
         failOnWarning="true"
         printerClass="\Drupal\Tests\Listeners\HtmlOutputPrinter"
         cacheResult="false"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.3/phpunit.xsd">
  <php>
    <ini name="error_reporting" value="32767"/>
    <ini name="memory_limit" value="-1"/>
    <env name="SIMPLETEST_BASE_URL" value="http://127.0.0.1:8080"/>
    <env name="SIMPLETEST_DB" value="mysql://root:root@mysql:3306/MODULE_NAME_testing"/>
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="/tmp/browser_output"/>
    <env name="SYMFONY_DEPRECATIONS_HELPER" value="weak"/>
    <env name="PHPUNIT_ASSERTIONS_DISABLE" value="assertionError"/>
    <env name="XDEBUG_MODE" value="off"/>
  </php>
  <testsuites>
    <testsuite name="unit">
      <directory>web/modules/custom/MODULE_NAME/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/MODULE_NAME/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/MODULE_NAME/tests/src/Functional</directory>
    </testsuite>
    <testsuite name="functional-javascript">
      <directory>web/modules/custom/MODULE_NAME/tests/src/FunctionalJavascript</directory>
    </testsuite>
  </testsuites>
  <listeners>
    <listener class="\Drupal\Tests\Listeners\DrupalListener">
    </listener>
  </listeners>
</phpunit>
```

## Custom Module Structure

Your module should follow the standard Drupal module structure, with tests in the appropriate directories:

```
MODULE_NAME/
├── MODULE_NAME.info.yml
├── MODULE_NAME.module
├── src/
│   └── ...
└── tests/
    └── src/
        ├── Functional/
        │   └── ModuleNameFunctionalTest.php
        ├── FunctionalJavascript/
        │   └── ModuleNameJsTest.php
        ├── Kernel/
        │   └── ModuleNameKernelTest.php
        └── Unit/
            └── ModuleNameUnitTest.php
```

## Using This Setup in a CI/CD Context

### Continuous Integration

This GitHub Actions workflow enables continuous integration by:

1. **Automated Testing**: Running tests automatically on every push and pull request
2. **Quality Assurance**: Ensuring your module works correctly before merging changes
3. **Early Bug Detection**: Identifying issues quickly in the development cycle

To enhance your CI pipeline:

* Add code linting (PHP_CodeSniffer, ESLint)
* Add static analysis (PHPStan, Psalm)
* Add code coverage reports (with tools like Codecov)

### Continuous Deployment

Extend this workflow for continuous deployment by:

1. Add a deployment step that triggers after successful tests on the main branch:

```yaml
# Add to your GitHub workflow file
- name: Deploy to development
  if: github.ref == 'refs/heads/main' && success()
  run: |
    # Add deployment scripts here
    ./scripts/deploy-to-dev.sh
```

2. Create deployment environments for different stages:

```yaml
name: Drupal Module CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    # Your testing job

  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/develop' && success()
    environment: development
    runs-on: ubuntu-latest
    steps:
      # Deployment steps

  deploy-prod:
    needs: test
    if: github.ref == 'refs/heads/main' && success()
    environment: production
    runs-on: ubuntu-latest
    steps:
      # Production deployment steps
```

## Troubleshooting

### Common Issues

1. **Web Server Connection Issues**:
   - Ensure the PHP server is running on 0.0.0.0 (all interfaces), not just localhost
   - Make sure the port (8080) is available
   - Check that SIMPLETEST_BASE_URL matches the server URL

2. **MySQL Connection Issues**:
   - Verify the database credentials match between Drupal installation and PHPUnit config
   - Check that the MySQL service is healthy

3. **Module Not Found**:
   - Ensure the module is correctly placed in web/modules/custom/
   - Verify the module name in Drush commands matches your module's machine name

4. **Windows-specific Issues**:
   - Always run Docker Desktop as Administrator
   - Run PowerShell as Administrator when executing act commands
   - Use the correct Docker socket path: `npipe:////./pipe/docker_engine`

## Further Resources

- [Drupal Testing Documentation](https://www.drupal.org/docs/automated-testing)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [act Documentation](https://github.com/nektos/act#readme)
