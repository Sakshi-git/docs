# Remove BLT from Acquia Multisite Setup

### Goal
> Remove BLT support from a Drupal 10 project hosted in Acquia cloud

### Introduction
BLT (Build and Launch Tool) is an Acquia tool which automates the process of testing, building and launching of an Drupal application. Acquia will end support for BLT on December 31, 2024.  Therefore here are the steps to update the Drupal implementation to remove the BLT support.

 ### 1. Remove BLT packages
   >- Ensure that your application is running on latest BLT release
   >- Ensure that your site is running normally without any errors
 ### 2. Replace BLT settings with Drupal Recommended Settings
 Remove the blt.settings.php include from your settings.php. Use Drupal Recommended Settings
 plugin, which provides same functionality as BLT.

````
composer require acquia/drupal-recommended-settings:dev-develop
````
### 3.Make the below changes in the code:
>- In the settings.php file, remove the following statement
```require DRUPAL_ROOT . "/../vendor/acquia/blt/settings/blt.settings.php”;```

>-  Update ```default.local.settings.php``` and  ```local.settings.php```
>>- Replace ```Acquia\Blt\Robo\Common\EnvironmentDetector
``` with ```Acquia\Drupal\RecommendedSettings\Helpers\EnvironmentDetector```
>>- Remove all traces on BLT/Behat in composer.json


### 4. Remove BLT files
Remove the BLT directory in your repo root: ```rm -rf blt```

### 5. Set up GrumPHP
>- Run ```composer require --dev phpro/grumphp php-parallel-lint/php-parallel-lint```
>- Accept the offer to create grumphp.yml in your project root folder.

```
parameters:
    git_dir: .
grumphp:
    ascii:
        failed: grumphp-grumpy.txt
        succeeded: grumphp-happy.txt
    process_timeout: 60
    tasks:
        git_commit_message:
            enforce_no_subject_trailing_period: false
            matchers:
            max_body_width: 0
            max_subject_width: 0
        yamllint: ~
        composer: ~
        phpcs:
            standard:
                - phpcs.xml.dist
                - vendor/drupal/coder/coder_sniffer/Drupal
                - vendor/drupal/coder/coder_sniffer/DrupalPractice
            ignore_patterns:
                - .github
                - .gitlab
                - bower_components
                - node_modules
                - vendor
            triggered_by:
                - php
                - module
        phplint: ~

```
### 6. Run the following commands to remove BLT from your project  and add Drush
>- ```composer remove acquia/blt```
>- ```composer require drush/drush```

### 7. Update the scripts in composer.json
>- Replace the BLT scripts with composer commands in scripts section in composer.json file

        ```
        {
            "drupal:install": "drush si minimal --existing-config",
            "drupal:test": "phpunit",
            "drupal:validate": "grumphp run"
        }
        ```
### 8. Update acquia-pipelines.yaml
>- Remove the BLT_DIR environment variable.
>- Replace call to setup_env script with: mysql -u root -proot -e "CREATE DATABASE IF NOT EXISTS drupal"
>- Replace call to validate script with: composer drupal:validate
>- Replace call to setup-app with: composer drupal:install
>- Replace call to tests with: composer drupal:test
>- Replace artifact build step with the following:
```
script:
  - mkdir /tmp/acli-push-artifact
  - acli push:artifact --dry-run --no-clone --no-commit --verbose --no-interaction --no-sanitize
  - mv ${SOURCE_DIR}/.git /tmp/acli-push-artifact/
  - shopt -s dotglob && rm -rf ${SOURCE_DIR}/* && mv /tmp/acli-push-artifact/* ${SOURCE_DIR}/

```
### 9. Verify changes
If frontend libraries are present in your source repository but missing from the build artifact, make sure they are properly defined as scaffold or vendor files in composer.json:

"docroot/themes/custom/{$name}": [

  "type:drupal-custom-theme"

],
### 10. Update cloud hooks
If you are using multiste setup , then you need to update the cloud hooks in your code.

>- Create the **hooks** folder in the project root
>- Create common **folder** under **hooks** directory
>- Create **post-code-update** under hooks/common
>- Add **post-code-update.sh** file under hooks/common/post-code-update
>- Add the following code to **post-code-update.sh** file
```
site="$1"
target_env="$2"
source_branch="$3"
deployed_tag="$4"
repo_url="$5"
repo_type="$6"


if [ "$target_env" != 'prod' ]; then
    echo "$site.$target_env: The $source_branch branch has been updated on $target_env. Clearing the cache."
    drush @$site.$target_env cr
else
    echo "$site.$target_env: The $source_branch branch has been updated on $target_env."
fi

```
No, Cloud Actions do not execute when the deployment occurs as a result of a commit update to a branch that is already deployed to a particular environment. The commit triggers an automatic deployment to the environment but that deployment does not trigger Cloud Actions.


The **post-code-update** will be called when the deployment occurs as a result of a commit update to a branch that is already deployed to a particular environment. The commit triggers an automatic deployment to the environment but that deployment does not trigger Cloud Actions.
Refer to [Cloud actions FAQ's](https://docs.acquia.com/acquia-cloud-platform/manage-apps/cloud-actions/faq)
