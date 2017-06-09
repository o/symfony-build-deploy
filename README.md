## Symfony2, Symfony3 build and deployment best practices 

#### Upload your code to the production server

Tag a version of your code as a release

    git tag ${TAG_NAME}
    git push --tags
        
Download a tagged version from Git as `tar.gz` archive
    
From any git server
    
    git archive --format=tar.gz -o "/path/to/archive/folder/${GIT_REPOSITORY}-${TAG_NAME}.tar.gz" --prefix="${GIT_REPOSITORY}-${TAG_NAME}/" ${TAG_NAME}
    
From Github

    curl -L --user "${GIT_USERNAME}:${GIT_PASSWORD}" --output "/path/to/archive/folder/${GIT_REPOSITORY}-${TAG_NAME}.tar.gz" "https://github.com/${GIT_ACCOUNT}/${GIT_REPOSITORY}/archive/${TAG_NAME}.tar.gz"

Create a release directory (Creating seperate folder for each release is strongly recommended)

    mkdir -p /path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}
    
Extract contents of archive to release directory

    tar -zxf "/path/to/archive/folder/${GIT_REPOSITORY}-${TAG_NAME}.tar.gz" --directory "/path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}" --strip-components 1

Copy parameters.yml contains production parameters to project folder

    cp "/path/to/parameters/${GIT_REPOSITORY}.yml" "/path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}/app/config/parameters.yml"

#### Post install tasks

    cd "/path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}"
    
For running `post-install-cmd` scripts run in the production environment

    export SYMFONY_ENV=prod
    
Install vendors

    composer.phar install --no-dev --optimize-autoloader
    
Clear cache

    /usr/bin/php app/console cache:clear --env=prod --no-debug
    
For Symfony3

    /usr/bin/php bin/console cache:clear --env=prod --no-debug
    
Dump your assets (If you need)
    
    /usr/bin/php app/console assetic:dump --env=prod --no-debug
    
For Symfony3

    /usr/bin/php bin/console assetic:dump --env=prod --no-debug
    
Executes (or dumps) the SQL needed to update the database schema to match the current mapping metadata.

    /usr/bin/php app/console doctrine:schema:update --force
    
For Symfony3

    /usr/bin/php bin/console doctrine:schema:update --force
    
Give necesssary permissions (in this example, www-data belongs to nginx)

    chown -R www-data: "/path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}"

Finally, symlink release folder to web server root

    ln -sf "/path/to/release/folder/${GIT_REPOSITORY}-${TAG_NAME}" "/var/www/${GIT_REPOSITORY}"

If you use op-code cache with `php-fpm`, restart `php5-fpm` for invalidating cache every release. If you're using Apache with `mod_php`, you should restart it.

** Don't forget to disable op-code cache on CLI interface using opcache.enable_cli (in php.ini) configuration option to prevent inconsistent results. **

#### Notes on deploying to multiple servers:

Compress files for easy transfer

    tar -zcf "/path/to/build/folder/${GIT_REPOSITORY}-${TAG_NAME}" .
    
Extract files to same directory on destination server

Following Ansible tasks helps to distribute `builds` to other servers

     - name: Create release directory
       file:
         path={{release_directory}}
         state=directory
 
     - name: Extracting release
       unarchive:
         src={{build_directory}}/{{project_name}}-{{release_version}}.tar.gz
         dest={{release_directory}}
         owner=www-data
 
     - name: Symlinking release
       file:
         src={{release_directory}}
         dest={{application_directory}}
         state=link
 
     - name: Start PHP5-FPM
       service:
         name=php5-fpm
         state=reloaded


