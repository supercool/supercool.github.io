---
layout: post
title:  "Migrating content within Craft"
date:   2016-01-22 13:00:00
author: Josh Angell
---

Recently as part of some site upgrades I’ve had a couple of jobs that have needed to copy some entries from one section in Craft to another, complete with content. I’m mainly putting this here for my own future reference but if it helps anyone else out then thats a bonus :)

There was a bunch of content in a Matrix that was the biggest issue and a few other non-relational fields. Crucially the original content needed to remain un-touched so I couldn’t just flip the `sectionId` of the elements in the database.

I’ve made heavy use of Tasks to do this so if you’ve never used Tasks before I’d suggest taking a look at Pixel & Tonics [PowerNap plugin](https://github.com/pixelandtonic/PowerNap) for a simpler overview of how they work in general.

Whilst this post covers moving Entries from one section to another for the sake of simplicity the method would be similar for any kind of content within Craft; for example you might need to migrate content from Entries to your own custom Element Type as I also had to do recently.

Also worth nothing is that I’ve used various IDs throughout the code - these can be found pretty easily by just browsing to the relevant page in the control panel and extracting the ID from the url.

## Task 1 - MigrationManager

Here we simply get the elements we want to migrate and run a sub Task for each batch.

```php
<?php

namespace Craft;

class MyPlugin_MigrateManagerTask extends BaseTask
{

  private $_elements;

  /**
   * @inheritDoc ITask::getDescription()
   *
   * @return string
   */
  public function getDescription()
  {
    return Craft::t('Migrating old content');
  }

  /**
   * @inheritDoc ITask::getTotalSteps()
   *
   * @return int
   */
  public function getTotalSteps()
  {
    // Setup the criteria for finding the elements we want to migrate
    $criteria = craft()->elements->getCriteria(ElementType::Entry);
    $criteria->enabled   = null;
    $criteria->limit     = null;
    $criteria->status    = null;
    $criteria->sectionId = 15; // The ID of the section we are copying from
    $elements = $criteria->find();

    // Chunk the elements into groups of 10 - if the content is quite
    // light you may want to up this to 100 or so
    $this->_elements = array_chunk($elements, 10);

    return count($this->_elements);
  }

  /**
   * @inheritDoc ITask::runStep()
   *
   * @param int $step
   *
   * @return bool
   */
  public function runStep($step)
  {
    // I frequently found I ran out of memory doing these sort of operations
    // so just bumped up what Craft is allowed to use here - in this to 2.5GB
    craft()->config->set('phpMaxMemoryLimit', '2560M');
    craft()->config->maxPowerCaptain();

    // Run the migration as a sub Task with the current chunk of elements
    return $this->runSubTask('MyPlugin_Migrate', null, array(
      'elements' => $this->_elements[$step]
    ));
  }

}
```



## Task 2 - Migrate

This Task handles the heavy lifting of duplicating content and saving the new Entry.

```php
<?php

namespace Craft;

class MyPlugin_MigrateTask extends BaseTask
{

  /**
   * @inheritDoc ITask::getTotalSteps()
   *
   * @return int
   */
  public function getTotalSteps()
  {
    return count($this->getSettings()->elements);
  }

  /**
   * @inheritDoc ITask::runStep()
   *
   * @param int $step
   *
   * @return bool
   */
  public function runStep($step)
  {
    // Migration logic
  }

  /**
   * @inheritDoc BaseSavableComponentType::defineSettings()
   *
   * @return array
   */
  protected function defineSettings()
  {

    return array(
      'elements' => AttributeType::Mixed
    );

  }

}
```


So, inside the `runStep()` method is where we migrate each element to the new section.

To start with I make sure we have enough memory and get the correct element from the settings array.

{% highlight php startinline %}
{% raw %}
// Again, bump the memory
craft()->config->set('phpMaxMemoryLimit', '2560M');
craft()->config->maxPowerCaptain();

// Get the element we want to copy from
$sourceElement = $this->getSettings()->elements[$step];
{% endraw %}
{% endhighlight %}


Next, check if the one we are copying to already exists or not. What you use to determine this will vary, in this case I just used the title but you may need something more bullet proof.

{% highlight php startinline %}
{% raw %}
$criteria = craft()->elements->getCriteria(ElementType::Entry);
$criteria->enabled   = null;
$criteria->limit     = null;
$criteria->status    = null;
$criteria->sectionId = 22; // The ID of the section we are copying to
$criteria->title     = $sourceElement->getContent()->title;
$targetElement       = $criteria->first();

// If we didn’t get an existing element, make one here
if (!$targetElement) {
  $targetElement = new EntryModel();
  $targetElement->sectionId = 22;
  $targetElement->typeId    = 23; // The ID of the Entry Type we want
}
{% endraw %}
{% endhighlight %}

Now we have the source and target elements sorted out we can copy across the field content - be aware that if the target element already exists and has content in the field then that content will be lost.

I had a large Matrix field that was the bulk of what I wanted to copy and it makes sense to deal with those first before sorting out the simpler fields. Taking my lead from [this StackExchange Q&A](https://craftcms.stackexchange.com/questions/8517/duplicating-matrix-fields-with-content-from-another-locale) I ended up with the following code to generate the Matrix Blocks:

{% highlight php startinline %}
{% raw %}
$newBlocks = array();
$i = 0;

foreach ($sourceElement->myMatrixField->find() as $block)
{
  // Setup a new block
  $newBlock = new MatrixBlockModel();
  $newBlock->fieldId = 4; // Whatever the ID of `myMatrixField` is
  $newBlock->typeId  = $block->getType()->id;
  $newBlock->ownerId = $targetElement->id;
  $newBlock->locale  = $block->locale;

  $newBlockContent = $newBlock->getContent();

  $values = array();

  // Loop the fields on this block
  foreach ($block->getFieldLayout()->getFields() as $blockFieldLayoutField)
  {
    $field = $blockFieldLayoutField->getField();
    $fieldHandle = $field->handle;

    // Cope with element fields by getting an array of their IDs
    if (in_array($field->type, array('Assets', 'Entries', 'Categories', 'Tags'))) {
      $value = $block->$fieldHandle->ids();
    } else {
      // For ‘normal’ fields just copy their content directly
      $value = $block->$fieldHandle;
    }

    $values[$fieldHandle] = $value;
  }

  // Set the content on the new block and add to the array
  $newBlock->setContentFromPost($values);
  $newBlocks['new'.$i] = $newBlock;
  $i++;
}
{% endraw %}
{% endhighlight %}

Once we have sorted out Matrix and anything else wild like [SuperTable](https://github.com/engram-design/SuperTable) (which should be pretty similar but don’t quote me on that) we can get on with setting what we’ve just done on the element along with any simpler content like so:

{% highlight php startinline %}
{% raw %}
$targetElement->setContent(array(
  'title' => $sourceElement->getContent()->title,

  // Here are the Matrix blocks we just made
  'myMatrixField' => $newBlocks,

  // Same as inside the Matrix, just get the IDs of relationship fields
  'someAssetField' => $sourceElement->someAssetField->ids(),

  // Simpler fields can just be directly copied
  'someSimpleTextField' => $sourceElement->someSimpleTextField,
));
{% endraw %}
{% endhighlight %}

Don’t forget to duplicate and attributes you may need, like `postDate` or whether the element is enabled or not:

{% highlight php startinline %}
{% raw %}
$targetElement->setAttributes(array(
  'slug'          => $sourceElement->slug,
  'postDate'      => $sourceElement->postDate,
  'expiryDate'    => $sourceElement->expiryDate,
  'enabled'       => $sourceElement->enabled,
  'archived'      => $sourceElement->archived,
  'localeEnabled' => $sourceElement->localeEnabled,
));
{% endraw %}
{% endhighlight %}

The final stage is just to save the element - in this case an Entry. I have wrapped the save method in a transaction in case anything goes wrong so we can catch and log errors without the Task hanging.

{% highlight php startinline %}
{% raw %}
$transaction = craft()->db->getCurrentTransaction() === null ? craft()->db->beginTransaction() : null;
try {

  // Try and save, throw an exception if it didn’t for some reason
  if (!craft()->entries->saveEntry($targetElement)) {

    // Try and get the errors so the log is more useful
    if ($targetElement->hasErrors()) {
      $firstError = array_shift($targetElement->getErrors())[0];
      throw new Exception(Craft::t('Couldn’t migrate from {title}. First error to correct: {error}', array('title' => $sourceElement->title, 'error' => firstError)));
    } else {
      throw new Exception(Craft::t('Couldn’t migrate from {title}.', array('title' => $sourceElement->title)));
    }

  }

  if ($transaction !== null)
  {
    $transaction->commit();
  }

} catch (Exception $e) {

  if ($transaction !== null)
  {
    $transaction->rollback();
  }

  // Log that exception message so we can debug it in the log viewer
  MyPlugin::log($e->getMessage(), LogLevel::Error);
  return $e->getMessage();

}

// Let the Task return true if nothing shifty happened above
return true;
{% endraw %}
{% endhighlight %}



## Fire it off

Now that we have both Tasks written we just need a way to create the manager Task and start it. There are a number of ways to do this but I prefer to make a simple controller action that does it:

{% highlight php startinline %}
{% raw %}
public function actionMigrate()
{
  // Create the Task
  craft()->tasks->createTask('MyPlugin_MigrateManager');

  if (!craft()->tasks->isTaskRunning()) {
    // Is there a pending task?
    $task = craft()->tasks->getNextPendingTask();

    if ($task) {
      // Attempt to close the connection if this is an Ajax request
      if (craft()->request->isAjaxRequest()) {
        craft()->request->close();
      }

      // Start running tasks
      craft()->tasks->runPendingTasks();
    }
  }
}
{% endraw %}
{% endhighlight %}

You can then either hit this action in your browser or call it via AJAX:

```bash
$ curl --silent -H \"X-Requested-With:XMLHttpRequest\" http://mysite.co.uk/actions/myPlugin/migrate
```

Thats it! You should now see the Tasks running and the content duplicating over to the new section.

I have put all the code used in this post together into a gist [here](https://gist.github.com/joshangell/26d6413a1eb243d4e8a1).
