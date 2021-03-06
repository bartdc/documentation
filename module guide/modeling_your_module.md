# Modeling your module

## What will the module do?

The easiest way to start developing a module is to make your mind up about what the module is supposed to be doing. You'll completely have to separate this for frontend and backend. In our example we'll talk about the frontend. The workflow for the backend would be identical, but we would do completely different things (delete, edit, add, ...) as we'll see later on.
In the frontend of our mini-blog there will be 4 different functionalities:

* View a list of our blogposts (Index)
* View the details of one blogpost (Detail)
* Get a list of the most recent posts (RecentPosts)
* Mark a blogpost as frikkin' awesome (ThisIsAwesome)

For every functionality you'll create a .php file.

## How will these functionalities work?

After you have decided what your module has to do, you have to decide when these will be executed:

* on pageload (see "how")
* **ajax**: when the page is already loaded

And how they will be presented

* **action**: the most important part of a page
* **widget**: just a smaller part of the page

Depending on these choices, you'll place your .php files in either the Actions, Ajax or Widgets folder.

As discussed earlier, actions are url-aware. When a module is linked to a page, the exact action that will be executed depends upon the url.
Given our previous example:

* *http://myforksite.local/**en**/mini-blog* has no action defined and will load the default action (in this case the list of our examples: Index.php)
* *http://myforksite.local/**en**/mini-blog/detail/fork-cms-ftw* defines an action detail, so detail.php will be loaded. It is up to Detail.php to determine what to do with the remaining fork-cms-ftw in the url.

For this reason, only 1 module can be linked to a page whereas several widgets (which are url-independent) can be linked to 1 page. Usually, widgets are lesser important parts that exist mainly to inform or attract users to actions.

In a blog module, the blogpost-overview and blogposts themselves obviously should be actions, whereas a widget could be a tiny part displaying the titles of the 3 latest blogposts. Such widget could be added to any page in the sidebar, the footer, …

The this_is_awesome action will be activated by clicking a button. This will add 1 to the awesomeness column of the record of a specific blog post. We will then hide that button, and update a counter on the page. Because the complete page will not to be reloaded, this is a typical AJAX-action.

> Combinations
> Sometimes it's easier to use one .php file for a couple of things you want to do. Imagine we would like to add a “comment” feature to the mini blog. The best way to do this is to add a comment form to the detail page (= add it to the comment action.) When the form is submitted, we reload the page and check if the form was submitted. If so, we add the comment and reload the detail page. This way of working is described in Chapter 11: Forms.

	 Note: You could of course be adding comments with Ajax too.

## Defining your module in the database

Before you can use the module, some records needs to be added to the database. Typically this is done by the installer you'll write at the end of this manual, but here's an overview already, so you can start developing your module.


### modules

In this table, you add the module name. Don't use camelCasing when giving the name, but the exact representation of the folder containing your module's code. *mini_blog* is what we add.

### modules_extras

The records in this table describe the different blocks and widgets we can add to our pages.
If the different actions of your module will be using the same page layout you can add a record which in our case looks something like:

| Column   | Value     | Additional information |
| -------- | --------- | ---------------------- |
| module   | MiniBlog  | The module name        |
| type     | block     | Either block or widget, depending on the choices made above
| label    | MiniBlog  | The label used to display the name of this page extra, this will call  lblMiniBlog - more on that in Chapter 10: Translations |
| action   | NULL      | Insert the exact name of the action (e.g. index, detail) if you only want this exact action to be shown when this is linked to a page (generally not advised, as every action will then have to be linked to another page). Insert NULL to let Fork CMS define the action based upon the url (advised, this way it is possible to make all actions of 1 module accessible through 1 page where this extra is linked to) |
| data     | NULL      | Extra data that has to available for the extra |
| hidden   | N         | Hide the extra from the overview |
| sequence | 7000      | The order of the widgets |

Should you want to add one specific action as a separate block - perhaps because the page layout (sidebar, footer, ...) is completely different from other actions so another template and thus another page must be used - you add the same record, but with the action field filled in, e.g. detail.

For our widget we need to add a record like this:

| Column   | Value        |
| -------- | ------------ |
| module   | MiniBlog     |
| type     | widget       |
| label    | RecentPosts  |
| action   | RecentPosts  |
| data     | NULL         |
| hidden   | N            |
| sequence | 7000         |


### groups_rights_modules

By default, only one group is installed (1: admin). If you want to give this group access to the module in the backend, you need to add a record for the module in this table.

Note: the user that installed Fork CMS is considered the God-user. This user always has all permissions. These rights are used for other users to determine if they have access to modules and actions in the backend. Rather than setting these rights directly into the database, we strongly recommend using the built-in Group Permissions tool (located at http://myforksite.local/private/en/groups/index) to set the correct privileges for your users, otherwise they will not be able to manage the module.

### groups_rights_actions

When you've given a group the rights for the module, you will need to add a record for each backend-action that the group must be able to use. In our case, we'll have an add, edit, delete and index action.

You'll see a field level in this table which you always must fill in with 7 (inspired by read, write & excecute in the UNIX filesystem)

Note: as described above, we strongly recommend using the built-in Group Permissions tool over manually adding the permissions in the database.

## The visible parts of the module

The action- & widget-files are controllers that will handle and manipulate the data, but do not contain any visual representation. For every action that has a visual representation we need to create a template.

As you might have already noticed, all the actions except the ThisIsAwesome action have a template-file (.tpl). They can all be found in the Layout/Templates folder, except the template for the RecentPosts widget. This can be found in the Layout/Widgets folder.

Templates files are regular .html-files but contain placeholders for the data the actionfiles will fetch from the database. More about these templates we'll see in Chapter 9: Templates.

When browsing (e.g. to http://myforksite.local/en/mini-blog/detail/fork-cms-ftw) the same routing conventions will be used to select the correct template. The template should therefor always have the same name as the action/widget file, followed by ".tpl"
If for any reason, you need to use a template that does not follow these naming conventions, it is possible to override this behaviour in the called-upon action/widget.

## Config

In the root folder of every module, there will be a Config.php. This is the starting point when the module is executed. Here you define which action is the default action (in case of no action is provided in the URL as we saw in the routing chapter), and which actions should be disabled.
> Why write disabled actions?
> You can disable actions in Config.php, but why would you write them in the first place?
> When you are writing your own modules, perhaps one day you'll find yourself in the situation where you can re-use an earlier written module (or use a module shared by someone else) but you don't need all it's actions. In that case, in the config file, you can disable these actions.
> Another plausible scenario is when you create actions that do bulk operations on the database, like populating the database with dummy content or removing all existing entries. You do not want the actions to accidently be called upon once the website goes live, but neither do you want to completely remove them (because you never know some day they might come in handy.)

## Backend navigation

For the frontend, you alter the complete navigation by dragging the page to the right places on the “pages”-page in the backend.

For the backend however, when you write a module, you need to add the backend navigation that is used to edit the module in the *backend_navigation* table. Actually, inserting the module into the backend navigation will usually be handled in the installer, which we'll get to in [Creating an installer](creating-an-installer).

For the mini-blog module, you'll want to add the following array to the element “children” of the array with the label “Modules”, after which the pages will become visible on the “Modules” page.

| column       | value                                                     |
| ------------ | --------------------------------------------------------- |
| id           | NULL                                                      |
| parent_id    | 2                                                         |
| label        | MiniBlog                                                  |
| url          | mini_blog/index                                           |
| selected_for | a:2:{i:0;s:13:"mini_blog/add";i:1;s:14:"mini_blog/edit";} |
| sequence     | 9                                                         |

Afterwards you'll have to delete the navigation cache file (/src/Backend/Cache/Navigation/navigation.php) to see the changes. And add a label `MiniBlog` to the backend core to have a nice text in your menu.
