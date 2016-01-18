---
layout: post
title:  "Migrating content within Craft"
date:   2016-01-14 10:00:00
author: Josh Angell
---

Recently as part of some site upgrades I’ve had a couple of jobs that have needed to copy some entries from one section in Craft to another, complete with content. I’m mainly putting this here for my own future reference but if it helps anyone else out then thats a bonus :)

There was a bunch of content in a Matrix that was the biggest issue and a few other non-relational fields. Crucially the original content needed to remain un-touched so I couldn’t just flip the `sectionId` of the elements in the database.

I’ve made heavy use of Tasks to do this so if you’ve never used Tasks before I’d suggest taking a look at Pixel & Tonics [PowerNap plugin](https://github.com/pixelandtonic/PowerNap/) for a simpler overview of how they work in general.

Also worth nothing is that I’ve used various IDs throughout the code - these can be found pretty easily by just browsing to the relevant page in the control panel and find the ID from url.

## Task 1 - MigrationManager

Here we simply get the elements we want to migrate and run a sub Task for each batch.

{% highlight php %}
{% raw %}
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
    $criteria->sectionId = 15;
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
{% endraw %}
{% endhighlight %}



## Task 2 - Migrate

This Task handles the heavy lifting of duplicating content and saving the new Entry.

{% highlight php %}
{% raw %}
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
{% endraw %}
{% endhighlight %}


So, inside the `runStep()` method we get the correct element from the settings array and migrate it to the new section.

{% highlight php %}
{% raw %}
// Again, bump the memory
craft()->config->set('phpMaxMemoryLimit', '2560M');
craft()->config->maxPowerCaptain();

// Get the element we want to copy from
$element = $this->getSettings()->elements[$step];
{% endraw %}
{% endhighlight %}


First, check if the one we are copying to already exists or not. What you use to determine this will vary, in this case I just used the title but you may need something more bullet proof.

{% highlight php %}
{% raw %}
$criteria = craft()->elements->getCriteria(ElementType::Entry);
$criteria->enabled   = null;
$criteria->limit     = null;
$criteria->status    = null;
$criteria->sectionId = 22;
$criteria->title     = $element->getContent()->title;
$targetElement       = $criteria->first();

// If we didn’t get an existing element, make one here
if (!$targetElement) {
  $targetElement = new EntryModel();
  $targetElement->sectionId = 22;
  $targetElement->typeId    = 23;
}
{% endraw %}
{% endhighlight %}

Next, copy across your field content - be aware that if the target element already exists and has content in the field then that content will be lost.

// Copy the blocks from a Matrix field -
// ref: https://craftcms.stackexchange.com/questions/8517/duplicating-matrix-fields-with-content-from-another-locale
$newBlocks = array();
$i = 0;

foreach ($element->myMatrixField->find() as $block)
{
  // Setup a new block
  $newBlock = new MatrixBlockModel();
  $newBlock->fieldId = 4;
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
      $value = $block->$fieldHandle;
    }

    $values[$fieldHandle] = $value;
  }

  // Set the content on the new block
  $newBlock->setContentFromPost($values);

  $newBlocks['new'.$i] = $newBlock;
  $i++;
}

// Set the content on the target element
$targetElement->setContent(array(
  'title' => $element->getContent()->title,
  'myMatrixField' => $newBlocks,

  // Same as before, just get the IDs of relationship fields
  'someAssetField' => $element->someAssetField->ids(),

  // Simpler fields can just be directly copied
  'someSimpleTextField' => $element->someSimpleTextField,
));

// Wrap in a transaction in case something goes wrong
$transaction = craft()->db->getCurrentTransaction() === null ? craft()->db->beginTransaction() : null;
try {

  // Try and save, throw an exception if it didn’t for some reason
  if (!craft()->entries->saveEntry($targetElement)) {

    // Try and get the errors so the log is more useful
    if ($targetElement->hasErrors()) {
      $firstError = array_shift($targetElement->getErrors())[0];
      throw new Exception(Craft::t('Couldn’t migrate from {title}. First error to correct: {error}', array('title' => $element->title, 'error' => firstError)));
    } else {
      throw new Exception(Craft::t('Couldn’t migrate from {title}.', array('title' => $element->title)));
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

return true;



TODO: Break code up into logical chunks and then provide a gist or zip of the
      files at the end (https://gist.github.com/joshangell/26d6413a1eb243d4e8a1)

TODO: Controller to fire it
