# The Entity System

In this chapter, we'll talk about how the bits and pieces of the entity work. We'll walk through what creating an entity type does behind the scenes, how bundles work, how Drupal loads and saves entities, and a bunch of other stuff.

But first, let's define some terms.

- **Entity types**: The highest level category of "thing" that you need to store data about. Examples from core include Node, User, Taxonomy Term, Taxonomy Vocabulary, Comment, and File.
- **Bundles**: Sub-categories of entity types. For example, the Node entity type can have bundles such as Article, Page, or Video.
- **Entities**: Specific instances of an entity type. For example, your latest blog post is an entity, meaning it's a specific instance of a "Blog Post" (bundle) which also makes it a "Node" (entity type).

As a simpler example, an "automobile" could be considered an entity type. It could have bundles such as "car", "boat", "plane", and "bike". Your '99 red Toyota Corolla would be an entity, i.e., a specific instance of the "car" bundle of the "automobile" entity type.

## A quick summary

## Creating an entity type

## Creating a bundle

## Loading an entity

## Saving an entity

## Deleting an entity