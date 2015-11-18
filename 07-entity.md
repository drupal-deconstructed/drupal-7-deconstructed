# The Entity System

**Note: this chapter is not yet complete!**

In this chapter, we'll talk about how the bits and pieces of the entity work. We'll walk through what creating an entity type does behind the scenes, how bundles work, how Drupal loads and saves entities, and a bunch of other stuff.

But first, let's define some terms.

- **Entity types**: The highest level category of "thing" that you need to store data about. Examples from core include Node, User, Taxonomy Term, Taxonomy Vocabulary, Comment, and File.
- **Bundles**: Sub-categories of entity types. For example, the Node entity type can have bundles such as Article, Page, or Video.
- **Entities**: Specific instances of an entity type. For example, your latest blog post is an entity, meaning it's a specific instance of a "Blog Post" (bundle) which also makes it a "Node" (entity type).

As a simpler example, an "automobile" could be considered an entity type. It could have bundles such as "car", "boat", "plane", and "bike". Your '99 red Toyota Corolla would be an entity, i.e., a specific instance of the "car" bundle of the "automobile" entity type.

## A quick summary

## Creating an entity type

Let's walk through what happens when we tell Drupal about our custom entity type. We'll be basing this on the code from [entity_example.module](https://api.drupal.org/api/examples/entity_example!entity_example.module/7). 

The first thing we would do to define our custom entity is to create the database tables needed for it. Here's what `entity_example.install` does, for example:

```php
function entity_example_schema() {
  $schema = array();

  // The name of the table can be any name we choose. However, namespacing the
  // table with the module name is best practice.
  $schema['entity_example_basic'] = array(
    'description' => 'The base table for our basic entity.',
    'fields' => array(
      'basic_id' => array(
        'description' => 'Primary key of the basic entity.',
        'type' => 'serial',
        'unsigned' => TRUE,
        'not null' => TRUE,
      ),
      // If we allow multiple bundles, then the schema must handle that;
      // We'll put it in the 'bundle_type' field to avoid confusion with the
      // entity type.
      'bundle_type' => array(
        'description' => 'The bundle type',
        'type' => 'text',
        'size' => 'medium',
        'not null' => TRUE,
      ),
      // Additional properties are just things that are common to all
      // entities and don't require field storage.
      'item_description' => array(
        'description' => 'A description of the item',
        'type' => 'varchar',
        'length' => 255,
        'not null' => TRUE,
        'default' => '',
      ),
      'created' => array(
        'description' => 'The Unix timestamp of the entity creation time.',
        'type' => 'int',
        'not null' => TRUE,
        'default' => 0,
      ),
    ),
    'primary key' => array('basic_id'),
  );

  return $schema;
}
```

That part should be pretty self-explanatory, since we've already talked about the bits and pieces at work there in the Database chapter.

The second step for us would be to implement [`hook_entity_info()`](https://api.drupal.org/api/drupal/modules%21system%21system.api.php/function/hook_entity_info/7) from a custom module. Our implementation will look something like this (taken from `entity_example.module`:

```php
function entity_example_entity_info() {
  $info['entity_example_basic'] = array(
    // A human readable label to identify our entity.
    'label' => t('Example Basic Entity'),

    // The controller for our Entity, extending the Drupal core controller.
    'controller class' => 'EntityExampleBasicController',

    // The table for this entity defined in hook_schema()
    'base table' => 'entity_example_basic',

    // Returns the uri elements of an entity.
    'uri callback' => 'entity_example_basic_uri',

    // IF fieldable == FALSE, we can't attach fields.
    'fieldable' => TRUE,

    // entity_keys tells the controller what database fields are used for key
    // functions. It is not required if we don't have bundles or revisions.
    // Here we do not support a revision, so that entity key is omitted.
    'entity keys' => array(
      // The 'id' (basic_id here) is the unique id.
      'id' => 'basic_id' ,
      // Bundle will be determined by the 'bundle_type' field.
      'bundle' => 'bundle_type',
    ),
    'bundle keys' => array(
      'bundle' => 'bundle_type',
    ),

    // FALSE disables caching. Caching functionality is handled by Drupal core.
    'static cache' => TRUE,

    // Bundles are alternative groups of fields or configuration
    // associated with a base entity type.
    'bundles' => array(
      'first_example_bundle' => array(
        'label' => 'First example bundle',
        // 'admin' key is used by the Field UI to provide field and
        // display UI pages.
        'admin' => array(
          'path' => 'admin/structure/entity_example_basic/manage',
          'access arguments' => array('administer entity_example_basic entities'),
        ),
      ),
    ),
    // View modes allow entities to be displayed differently based on context.
    // As a demonstration we'll support "Tweaky", but we could have and support
    // multiple display modes.
    'view modes' => array(
      'tweaky' => array(
        'label' => t('Tweaky'),
        'custom settings' => FALSE,
      ),
    ),
  );

  return $info;
}
```

With those two steps (the DB tables and the implementation of `hook_entity_info()` have just provided Drupal with a bunch of info about our custom entity, which we are calling `entity_example_basic`. But what does Drupal do with it all?

The answer is that Drupal doesn't really do anything immediately. There's no big "DISCOVER MY NEW ENTITY TYPE" button that you have to click to tell Drupal to find your custom entity type. All that you've done is set up the database tables and made the entity type info available for when it does need to be requested. Eventually, something in Drupal core will call [`entity_get_info()`](https://api.drupal.org/api/drupal/includes%21common.inc/function/entity_get_info/7)...WTF HAPPENS HERE??

## Creating bundles

## Loading an entity

## Saving an entity

## Deleting an entity