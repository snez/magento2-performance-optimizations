# Magento 2 performance optimizations

Magento 2 performance optimizations aimed for developers actively developing for Magento 2

The idea is to run Magento 2 in production mode for performance, but make adjustments to it so that it does not need to be recompiled and the static files do not need to be redeployed with every change. Additionally, some further adjustments will be necessary to display debugging errors and allow symlinks for module development.

Enable production mode:

`php bin/magento deploy:mode:set production`


Enable all caches:

`php bin/magento cache:enable`

Increase error verbosity #1:

```patch
diff --git a/app/bootstrap.php b/app/bootstrap.php
index 6701a9f..fac9b82 100644
--- a/app/bootstrap.php
+++ b/app/bootstrap.php
@@ -8,7 +8,7 @@
  * Environment initialization
  */
 error_reporting(E_ALL);
-#ini_set('display_errors', 1);
+ini_set('display_errors', 1);
 
 /* PHP version validation */
 if (!defined('PHP_VERSION_ID') || !(PHP_VERSION_ID === 70002 || PHP_VERSION_ID === 70004 || PHP_VERSION_ID >= 70006)) {
 ```

Increase error verbosity #2:

```patch
diff --git a/pub/errors/processor.php b/pub/errors/processor.php
index 5ca9d82..ee32424 100644
--- a/pub/errors/processor.php
+++ b/pub/errors/processor.php
@@ -332,7 +332,7 @@ class Processor
 
         //initial settings
         $config = new \stdClass();
-        $config->action         = '';
+        $config->action         = 'print';
         $config->subject        = 'Store Debug Information';
         $config->email_address  = '';
         $config->trash          = 'leave';
```

Increase error verbosity #3:

```patch
diff --git a/vendor/magento/framework/Webapi/ErrorProcessor.php b/vendor/magento/framework/Webapi/ErrorProcessor.php
index bb86c6b..edc1f97 100644
--- a/vendor/magento/framework/Webapi/ErrorProcessor.php
+++ b/vendor/magento/framework/Webapi/ErrorProcessor.php
@@ -108,7 +108,7 @@ class ErrorProcessor
      */
     public function maskException(\Exception $exception)
     {
-        $isDevMode = $this->_appState->getMode() === State::MODE_DEVELOPER;
+        $isDevMode = true;//$this->_appState->getMode() === State::MODE_DEVELOPER;
         $stackTrace = $isDevMode ? $exception->getTraceAsString() : null;
 
         if ($exception instanceof WebapiException) {
```

Increase error verbosity #4:

```patch
diff --git a/vendor/magento/framework/App/StaticResource.php b/vendor/magento/framework/App/StaticResource.php
index 87a2c37..a4ee009 100644
--- a/vendor/magento/framework/App/StaticResource.php
+++ b/vendor/magento/framework/App/StaticResource.php
@@ -143,7 +143,7 @@ class StaticResource implements \Magento\Framework\AppInterface
     public function catchException(Bootstrap $bootstrap, \Exception $exception)
     {
         $this->getLogger()->critical($exception->getMessage());
-        if ($bootstrap->isDeveloperMode()) {
+        if (true || $bootstrap->isDeveloperMode()) {
             $this->response->setHttpResponseCode(404);
             $this->response->setHeader('Content-Type', 'text/plain');
             $this->response->setBody($exception->getMessage() . "\n" . $exception->getTraceAsString());
```

When we deploy static assets, instead of copying the files, symlink them instead to the originals, so that we do not need to re-deploy them while we are developing the modules:

```patch
diff --git a/app/etc/di.xml b/app/etc/di.xml
index e17505e..193d166 100644
--- a/app/etc/di.xml
+++ b/app/etc/di.xml
@@ -643,7 +643,8 @@
         <arguments>
             <argument name="strategiesList" xsi:type="array">
                 <item name="view_preprocessed" xsi:type="object">Magento\Framework\App\View\Asset\MaterializationStrategy\Symlink</item>
-                <item name="default" xsi:type="object">Magento\Framework\App\View\Asset\MaterializationStrategy\Copy</item>
+                <item name="default" xsi:type="object">Magento\Framework\App\View\Asset\MaterializationStrategy\Symlink</item>
+                <item name="asset" xsi:type="object">Magento\Framework\App\View\Asset\MaterializationStrategy\Copy</item>
             </argument>
         </arguments>
     </virtualType>
```

Because we will be using symlinks, we now have to adjust Magento to skip file validation which is normally there for security purposes (i.e. don't do anything stupid on an actual live server). This will also allow us to use modman to symlink modules into the codebase.

```patch
diff --git a/vendor/magento/framework/View/Element/Template/File/Validator.php b/vendor/magento/framework/View/Element/Template/File/Validator.php
index 4b4e7e1..8dc7a5f 100644
--- a/vendor/magento/framework/View/Element/Template/File/Validator.php
+++ b/vendor/magento/framework/View/Element/Template/File/Validator.php
@@ -103,6 +103,7 @@ class Validator
      */
     public function isValid($filename)
     {
+        return true;
         $filename = str_replace('\\', '/', $filename);
         if (!isset($this->_templatesValidationResults[$filename])) {
             $this->_templatesValidationResults[$filename] =
```

Also because we are not redeploying the assets, have magento generate them on the fly without refering to a deployment version:

```patch
diff --git a/vendor/magento/framework/App/View/Deployment/Version.php b/vendor/magento/framework/App/View/Deployment/Version.php
index 67f6d3c..393a242 100644
--- a/vendor/magento/framework/App/View/Deployment/Version.php
+++ b/vendor/magento/framework/App/View/Deployment/Version.php
@@ -79,7 +79,7 @@ class Version
     {
         $result = $this->versionStorage->load();
         if (!$result) {
-            if ($appMode == \Magento\Framework\App\State::MODE_PRODUCTION
+            if (false && $appMode == \Magento\Framework\App\State::MODE_PRODUCTION
                 && !$this->deploymentConfig->getConfigData(
                     ConfigOptionsListConstants::CONFIG_PATH_SCD_ON_DEMAND_IN_PRODUCTION
                 )
```

and


```patch
diff --git a/vendor/magento/framework/App/StaticResource.php b/vendor/magento/framework/App/StaticResource.php
index 87a2c37..2871dca 100644
--- a/vendor/magento/framework/App/StaticResource.php
+++ b/vendor/magento/framework/App/StaticResource.php
@@ -116,7 +116,7 @@ class StaticResource implements \Magento\Framework\AppInterface
     {
         // disabling profiling when retrieving static resource
         \Magento\Framework\Profiler::reset();
-        $appMode = $this->state->getMode();
+        $appMode = \Magento\Framework\App\State::MODE_DEVELOPER; // $this->state->getMode();
         if ($appMode == \Magento\Framework\App\State::MODE_PRODUCTION
             && !$this->deploymentConfig->getConfigData(
                 ConfigOptionsListConstants::CONFIG_PATH_SCD_ON_DEMAND_IN_PRODUCTION
```

Now if someone could put the above into a Magento 2 module...
