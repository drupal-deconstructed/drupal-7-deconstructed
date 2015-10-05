# Nodes

**Note: this chapter is not yet complete!**

Nodes are Drupal's most basic content unit. Traditionally in Drupal every type of content like a page or blog post is a node, although in Drupal 7 it is not strictly necessary to use nodes when something like a Views Page or an Entity record might be more suitable. Modules can define their own node types as well.

Here are the <a href="https://www.drupal.org/node/49768">default properties of Drupal 7 node objects from the official documentation</a>. Generally all these properties are available in the theme layer when working with node displays:

<table>
  <tr>
    <td>$node->nid</td>
    <td>Node ID.</td>
  </tr>
  <tr>
    <td>$node->vid</td>
    <td>The revision ID of the current version of this node.</td>
  </tr>
  <tr>
    <td>$node->type</td>
    <td>Type of node (e.g. book, page, forum), which is also the entity bundle.</td>
  </tr>
  <tr>
    <td>$node->language</td>
    <td>The default language for this node.</td>
  </tr>
  <tr>
    <td>$node->title</td>
    <td>Page (or, more accurately, node) title.</td>
  </tr>
  <tr>
    <td>$node->uid</td>
    <td>User ID of node creator.</td>
  </tr>
  <tr>
    <td>$node->status</td>
    <td>unpublished/published (NODE_NOT_PUBLISHED | NODE_PUBLISHED).</td>
  </tr>
  <tr>
    <td>$node->created</td>
    <td>UNIX timestamp of node creation date.</td>
  </tr>
  <tr>
    <td>$node->changed</td>
    <td>UNIX timestamp of last time node was changed.</td>
  </tr>
  <tr>
    <td>$node->comment</td>
    <td>Whether comments are allowed on this node (COMMENT_NODE_HIDDEN | COMMENT_NODE_CLOSED |  COMMENT_NODE_OPEN)</td>
  </tr>
  <tr>
    <td>$node->promote</td>
    <td>Promoted to front page (NODE_NOT_PROMOTED | NODE_PROMOTED).</td>
  </tr>
  <tr>
    <td>$node->sticky</td>
    <td>Sticky (NODE_NOT_STICKY | NODE_STICKY).</td>
  </tr>
  <tr>
    <td>$node->tnid</td>
    <td>The node ID of the translation source (or parent, if node is a translation).</td>
  </tr>
  <tr>
    <td>$node->translate</td>
    <td>Does the translation need to be updated (0|1)?</td>
  </tr>
  <tr>
    <td>$node->revision_uid</td>
    <td>The user ID of the user who created the current revision.</td>
  </tr>
  <tr>
    <td>$node->body</td>
    <td>Array. Body content of node. Long text field with summary.<br /><em>Note: Don't assume that this field will exist, as it is possible to remove it via <code>Manage Fields</code> on each content type. Similarly, modules that define a custom node content type may not even attach a body in the first place.</em></td>
  </tr>
  <tr>
    <td>$node->log</td>
    <td>Message left by the creator of this revision, explaining the changes.</td>
  </tr>
  <tr>
    <td>$node->revision_timestamp</td>
    <td>Unix timestamp showing when current revision was created.</td>
  </tr>
  <tr>
    <td>$node->name</td>
    <td>Username of node creator.</td>
  </tr>
  <tr>
    <td>$node->picture</td>
    <td>User avatar of the node creator.</td>
  </tr>
  <tr>
    <td>$node->cid</td>
    <td>CID of last comment?</td>
  </tr>
  <tr>
    <td>$node->last_comment_timestamp</td>
    <td>Timestamp of last comment (Unix Epoch C).</td>
  </tr>
  <tr>
    <td>$node->last_comment_name</td>
    <td>Name of last comment author</td>
  </tr>
  <tr>
    <td>$node->last_comment_uid</td>
    <td>UID of last comment author.</td>
  </tr>
  <tr>
    <td>$node->comment_count</td>
    <td>Number of comments made on node.</td>
  </tr>
  <tr>
    <td>$node->data</td>
    <td>Serialized string of data associated with the node.</td>
  </tr>
  <tr>
    <td>$node->rdf_mapping</td>
    <td>W3C standard to describe structured data. See http://api.drupal.org/api/drupal/modules!rdf!rdf.module/group/rdf/7</td>
  </tr>
</table>

## Adding properties to nodes

To <a href="http://drupal.stackexchange.com/questions/78325/how-to-create-a-new-properties-to-node-or-user-and-so-on">create a new property on a node</a> programmatically, the <a href="https://drupal.org/project/entity">Entity API</a> module can be very helpful. The <a href="http://www.drupalcontrib.org/api/drupal/contributions!entity!entity.api.php/function/hook_entity_property_info/7">hook_entity_property_info</a> hook lets you set up the metadata properties. The <a href="http://www.drupalcontrib.org/api/drupal/contributions!entity!modules!node.info.inc/function/entity_metadata_node_entity_property_info/7">entity_metadata_node_entity_property_info</a> function implements hook_entity_property_info() on top of the node module, for default node property definitions.

You can find more documentation in the <a href="http://drupal.org/node/878784">Entity API handbooks</a>. Also check the <a href="http://drupalcode.org/project/entity.git/blob/refs/heads/7.x-1.x:/README.txt">README</a> and the provided API docs in <a href="http://drupalcode.org/project/entity.git/blob/refs/heads/7.x-1.x:/entity.api.php"><code>entity.api.php</code></a>.
