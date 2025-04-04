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
* Selenium server for JavaScript tests

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

## Running Tests with GitHub Actions Locally

To run ALL tests using GitHub Actions workflow locally:

### Windows

1. Start Docker Desktop as Administrator

2. Run this command in PowerShell as Administrator:

```powershell
act -W .github/workflows/phpunit.yml -j phpunit -P ubuntu:ghcr.io/catthehacker/ubuntu:act-latest --container-daemon-socket "npipe:////./pipe/docker_engine"
``` 

### Linux/macOS

1. Run act:

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
    branches:
      - main
      - develop
  pull_request:

jobs:
  phpunit:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:5.7
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: drupal_testing
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
      selenium:
        image: selenium/standalone-chrome:latest
        ports:
          - 4444:4444

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.2
          extensions: gd, pdo_mysql, mbstring
          coverage: none

      - name: Install Composer dependencies
        run: |
          composer install --no-interaction --prefer-dist
          cp web/sites/default/default.settings.php web/sites/default/settings.php
          composer require drush/drush --no-interaction

      - name: Install Drupal
        env:
          SIMPLETEST_DB: "mysql://root:root@127.0.0.1/drupal_testing"
          SIMPLETEST_BASE_URL: "http://127.0.0.1"
        run: |
          # Use Drupal's test site install script (adjust the path if needed)
          php web/core/scripts/test-site.php install --db-url="$SIMPLETEST_DB" --base-url="$SIMPLETEST_BASE_URL" --install-profile=standard --langcode=en

      - name: Enable required modules
        run: |
          chmod +x ./vendor/bin/drush
          chmod +x ./vendor/drush/drush/drush
          chmod +x ./vendor/bin/drush.php
          
          # Enable module with debug output
          php ./vendor/drush/drush/drush en draggable_mapper -y

      - name: Prepare testing environment
        run: |
          # Create all parent directories with proper permissions
          mkdir -p web/sites/simpletest/browser_output
          
          # Set world-writable permissions recursively
          chmod -R 777 web/sites

      - name: Run PHPUnit
        env:
          BROWSERTEST_SELENIUM_URL: "http://localhost:4444"
          MINK_DRIVER_ARGS_WEBDRIVER: '["chrome", {"browserName": "chrome", "platformName": "LINUX", "goog:chromeOptions": {"args": ["--disable-gpu", "--headless", "--no-sandbox", "--disable-dev-shm-usage"]}}, "http://localhost:4444"]'
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
          
          # Return to root directory before running tests
          cd ..
          
          # Run tests
          export BROWSERTEST_OUTPUT_DIRECTORY="$(pwd)/web/sites/simpletest/browser_output"
          chmod +x ./vendor/bin/phpunit
          ./vendor/bin/phpunit -c $(pwd)/phpunit.xml --verbose --debug --log-junit test-results.xml
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
    <env name="SIMPLETEST_DB" value="mysql://root:root@127.0.0.1/drupal_testing"/>
    <env name="BROWSERTEST_OUTPUT_DIRECTORY" value="web/sites/simpletest/browser_output"/>
    <env name="BROWSERTEST_SELENIUM_URL" value="http://localhost:4444"/>

    // minK CLASS??
    <env name="SYMFONY_DEPRECATIONS_HELPER" value="weak"/>
    <env name="PHPUNIT_ASSERTIONS_DISABLE" value="assertionError"/>
    <env name="XDEBUG_MODE" value="off"/>
  </php>
  <testsuites>
    <testsuite name="unit">
      <directory>web/modules/custom/draggable_mapper/tests/src/Unit</directory>
    </testsuite>
    <testsuite name="kernel">
      <directory>web/modules/custom/draggable_mapper/tests/src/Kernel</directory>
    </testsuite>
    <testsuite name="functional">
      <directory>web/modules/custom/draggable_mapper/tests/src/Functional</directory>
    </testsuite>
    <testsuite name="functional-javascript">
      <directory>web/modules/custom/draggable_mapper/tests/src/FunctionalJavascript</directory>
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

5.

## Further Resources

- [Drupal Testing Documentation](https://www.drupal.org/docs/automated-testing)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [act Documentation](https://github.com/nektos/act#readme)
