title: Howto generate entities from table with Doctrine
date: 2015-08-31 23:55:59
tags:
- php
- doctrine
---

1. Generate xml from table.
   
   - you may need to set filter in your cli-config.php code like below.
     
     ``` php
     $config->setFilterSchemaAssetsExpression('/^life_/');
     ```
   - you may meet unkown database type enum requested error, you can fix that in your cli-config.php too.
     
     ``` php
     $entityManager->getConnection()->getDatabasePlatform()->registerDoctrineTypeMapping('enum', 'string');
     ```
<!--more-->
2. Generate php entity class from xml.
   
   - you should set your configuration as XmlMetadataConfiguration as below
     
     ``` php
     $config = \Doctrine\ORM\Tools\Setup::createXmlMetadataConfiguration($entitesPath, $dev);
     ```
   - then execute the command below
     
     ``` shell
     php vendor/bin/doctrine.php orm:convert-mapping annotation  ./src/Fudou/Entity
     ```
3. All above steps are not necessary. In fact we just need the following command. And it will generate directories for you as you specified the namespace.
   ``` shell
   php vendor/bin/doctrine.php orm:convert-mapping --from-database --namespace='Fudou\Entity\' 
   annotation ./src/
   ```
