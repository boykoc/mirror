pipeline {
    agent any
    stages {
        stage('Preperation') {
            steps {
                sh 'echo "Preparing environment..."'
                sh 'sudo apt-get install -y python-dev postgresql libpq-dev python-pip python-virtualenv git-core solr-jetty openjdk-8-jdk redis-server'
                sh 'echo "Jetty failing due to jenkins on port 8080, switching to 8983"'
                sh 'sudo sed -i -e "s/JETTY_PORT=8080/JETTY_PORT=8983/g" /etc/default/jetty8'
                sh 'sudo cat /etc/default/jetty8'
                sh 'sudo apt-get install -y python-dev postgresql libpq-dev python-pip python-virtualenv git-core solr-jetty openjdk-8-jdk redis-server'
                
                sh 'echo "Symlink home directory..."'
                sh 'mkdir -p ~/ckan/lib'
                sh 'sudo ln -sf ~/ckan/lib /usr/lib/ckan'
                sh 'mkdir -p ~/ckan/etc'
                sh 'sudo ln -sf ~/ckan/etc /etc/ckan'
                
                sh 'echo "Create virtualenv"'
                sh 'sudo mkdir -p /usr/lib/ckan/default'
                sh 'sudo chown `whoami` /usr/lib/ckan/default'
                sh 'virtualenv --no-site-packages /usr/lib/ckan/default'
                sh '. /usr/lib/ckan/default/bin/activate'
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    pip install setuptools==36.1
                    pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.8.2#egg=ckan'
                    pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt
                    
                    deactivate
                    . /usr/lib/ckan/default/bin/activate
                   '''
            }
        }
        stage('SetupPostgreSQL') {
            steps {
                sh 'echo "Seting up PostgreSQL Database..."'
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    sudo -u postgres psql -l
                    sudo -u postgres psql --command="CREATE USER ckan_default WITH PASSWORD 'pass' NOSUPERUSER NOCREATEDB NOCREATEROLE;"
                    sudo -u postgres createdb -O ckan_default ckan_default -E utf-8

                    paster datastore set-permissions -c test-core.ini | sudo -u postgres psql
                    

                   '''
            }
        }
        stage('CreateCKANConfig') {
            steps {
                sh 'echo "Creating CKAN config file..."'
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    sudo mkdir -p /etc/ckan/default
                    sudo chown -R `whoami` /etc/ckan/
                    sudo chown -R `whoami` ~/ckan/etc
                    
                    paster make-config ckan /etc/ckan/default/development.ini
                    cat /etc/ckan/default/development.ini
                    sudo sed -i -e "s+ckan.site_url =+ckan.site_url = http://jenkins584398.canadacentral.cloudapp.azure.com+g" /etc/ckan/default/development.ini
                   '''
            }
        }
        stage('SetupSolr') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    sudo sed -i -e "s+#JETTY_HOST=\$(uname -n)+JETTY_HOST=127.0.0.1+g" /etc/default/jetty8
                    sudo service jetty8 restart
                    sudo mv /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
                    sudo ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/conf/schema.xml
                    sudo service jetty8 restart
                   '''
            }
        }
        stage('LinkWHO') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini
                 '''
            }
        }
        stage('CreateDBTables') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    cd /usr/lib/ckan/default/src/ckan
                    paster db init -c /etc/ckan/default/development.ini
                   '''
            }
        }
        stage('SetupDataStore') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    sed -i '/^ckan.plugins = / s/$/ datastore/' /etc/ckan/default/development.ini
                    sudo -u postgres psql -l
                    sudo -u postgres psql --command="CREATE USER datastore_default WITH PASSWORD 'pass' NOSUPERUSER NOCREATEDB NOCREATEROLE;"
                    sudo -u postgres createdb -O ckan_default datastore_default -E utf-8
                    sed -i '/#ckan.datastore.write_url = /s/^#//g' /etc/ckan/default/development.ini
                    sed -i '/#ckan.datastore.read_url/s/^#//g' /etc/ckan/default/development.ini
                    
                    paster --plugin=ckan datastore set-permissions -c /etc/ckan/default/development.ini | sudo -u postgres psql --set ON_ERROR_STOP=1
                    
                   '''
            }
        }
        stage('SetupTestingEnvironment') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    pip install -r /usr/lib/ckan/default/src/ckan/dev-requirements.txt
                    
                    sudo -u postgres createdb -O ckan_default ckan_test -E utf-8
                    sudo -u postgres createdb -O ckan_default datastore_test -E utf-8
                    
                    cd /usr/lib/ckan/default/src/ckan/
                    paster datastore set-permissions -c test-core.ini | sudo -u postgres psql
                    
                    sudo cp -r /usr/share/solr/./ /etc/solr/ckan
                    sudo sed -i '/#ckan.datastore.read_url/s/^#//g' /etc/ckan/default/development.ini
                    sudo sed -i \'/    <core name="collection1" instanceDir="." \\/>/a\\ \\ \\ \\ <core name="ckan" instanceDir="/etc/solr/ckan" \\/>\' /usr/share/solr/./solr.xml
                    sudo service jetty8 restart
                    sed -i \'/^solr_url = / s/$/\\/ckan/\' /etc/ckan/default/development.ini
                   '''
            }
        }
        stage('OntarioThemeTests') {
            steps {
                sh '''
                    
                    cd /usr/lib/ckan/default/src/ckan/ckanext
                    '''
                checkout scm
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    cd ckanext-ontario_theme
                    python setup.py develop
                    pip install -r dev-requirements.txt
                    sed -i '/^ckan.plugins = / s/$/ ontario_theme/' /etc/ckan/default/development.ini
                    
                    cd ../
                    git clone https://github.com/ckan/ckanapi-exporter.git
                    cd ckanapi-exporter
                    python setup.py develop
                    pip install -r dev-requirements.txt
                    
                    cd ../ckanext-ontario_theme
                    nosetests --with-pylons=test.ini --with-xunit --xunit-file=${WORKSPACE}/nosetests.xml
                    ls -lah
                   '''
            }
            post {
                always {
                    xunit thresholds: [failed(failureThreshold: '100', unstableThreshold: '100')], tools: [Custom(customXSL: '${JENKINS_HOME}/workspace/custom-to-junit.xsl', deleteOutputFiles: true, failIfNotNew: true, pattern: '**/nosetests.xml', skipNoTestFiles: false, stopProcessingIfError: true)]
                }
            }
        }
        stage('RunTests') {
            steps {
                sh '''
                    . /usr/lib/ckan/default/bin/activate
                    which pip
                    
                    
                   '''
            }
        }        
    }
    post {
        always {
            echo 'One way or another, I have finished'
            sh '''
                echo "Removing packages and CKAN files for clean builds in the future..."
             
                sudo -u postgres dropdb ckan_test
                sudo -u postgres dropdb datastore_test
                sudo -u postgres dropdb ckan_default
                sudo -u postgres dropdb datastore_default
                
                sudo -u postgres dropuser datastore_default
                sudo -u postgres dropuser ckan_default
                
                
                sudo apt-get remove --purge -y --allow-downgrades --allow-remove-essential --allow-change-held-packages python-dev libpq-dev python-pip python-virtualenv git-core solr-jetty openjdk-8-jdk redis-server
                sudo apt autoremove -y
                
                sudo rm -rf /usr/lib/ckan
                sudo rm -rf /etc/ckan
                sudo rm -rf ckan
                sudo rm -rf /var/lib/jenkins/ckan
               '''            
        }
        success {
            echo 'I succeeeded!'
        }
        unstable {
            echo 'I am unstable :/'
        }
        failure {
            echo 'I failed :('
        }
        changed {
            echo 'Things were different before...'
        }
    }    
}