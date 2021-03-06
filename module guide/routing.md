# Routing

One of the strong points of Fork is it's ability to maintain very readable URL's. Not only is this
userfriendly, it's very interesting from an SEO point of view too.


## Existing file or folder

When any URL is accessed on your domain, the .htaccess file of Fork CMS is consulted. The most
important thing it does is checking if the URL which was opened, is a physical location (file or folder).
If this is the case, that location is opened. If the requested url is not an existing file or folder,
the routing begins, follow the rules below.


## Application

If the URL isn't a physical location, a rewrite will occur to the frontcontroller (/index.php`).
This frontcontroller will create an AppKernel, which will kickstart our application router. The
router will determine which application should be initiated. The following rules are followed:

* `/private` or `/backend` => backend application
* `/api`	=> API application
* `/install` => install application
* `/*` => frontend application

The router will then initiate the correct application.


## Language

The frontend and backend application will check if you are running a multilanguage website or not.
When installing Fork CMS you had to specify which languages should be installed. In this case a
language indicator will be present in the url.

> Multilanguage
> If you made a mistake while installing Fork as a multilingual site, you can simply change this by
> altering the SITE_MULTILANGUAGE -global in the `app/config/parameters.yml` file.
> Caution: this will alter the generated urls, so pages indexed by search engines may result in a 404.

> Active languages
> To handle the management of content of different languages, you can enable/disable languages for
> the frontend. This can be done on the Settings page in Fork CMS.


If your site is multilingual the first part of the URL is interpreted as a language abbreviation:
http://myforksite.dev/en/mini-blog/detail/fork-cms-ftw.

If this is an active language, that language is chosen.

> No language provided
> If your site is multilingual but the the first element is not evaluated as a valid language
> (e.g. myforksite.local or myforksite.local/mini-blog), Fork CMS tries to determine the language
> based upon the following checks:
> * A language variable exists in the session (from a previous visit)
> * A language variable exists in a cookie (from a previous visit)
> * The user's browser language matches an active language
> * Fallback: the default language defined in the `/app/config/parameters.yml` file


## Page

After determining the language each application will try and map the URL to a module and action(s).
Frontend and backend handle this in a different way.


### Frontend

Which routes (URL's) are available is determined by the pages module. This module will generated a
cache file(s) containing all possible routes and which modules/actions are linked to that route. These
cache file(s) are saved per language:

* `/src/Frontend/Cache/Navigation/keys_en.php`
* `/src/Frontend/Cache/Navigation/navigation_en.php`

These cache files are volatile and should never by modified manually.

In the case of `http://fork.dev/en/mini-blog/detail/pigs-ftw`, a search will start for *mini-blog/detail/pigs-ftw*.
If this string is not found, the last slug of the url is omitted (*mini-blog/detail/pigs-ftw*
becomes *mini-blog/detail*) and the cache file is searched again. This happens over and over until
a page is found. If no page is found, the request will return the 404 page.

> Nesting of pages
> When a page is located in a nested structure, this will be visible in the URL. E.g. in the
> standard installation of Fork CMS, the “Location” page is located beneath “About-us”.
> The URL for this page looks like: http://myforksite.dev/en/about-us/location
> It's the part in bold that is saved in the cache file.

#### Action

If a page is found (*mini-blog*), the next part of the URL (if any) will be interpreted as an action,
(in the case of *mini-blog/detail/pigs-ftw*, the action will be detail). If this is an existing action
in the module linked to the page, that action will be executed. If it doesn't exist, the default
action defined in the config-file of the module will be executed.

The action-part in the url (in this case: detail) will map to a matching [action-translation](translations-and-locale).
*detail* is one of the default actions already, but if you're adding a new or uncommon action, don't
forget to create a translation for it. Internally, the action-part in the url will be searched for
in the translations, and it's reference code will be used to route to the correct action, this way
it's possible to even translate that part in the url when dealing with multiple languages.

Example: http://myforksite.local/en/mini-blog/detail/pigs-ftp

Fork CMS will browse through all action-translations for this language (in this case: en) until it
finds one that's been translated to detail. Here, it'll find the action translation with reference
code Detail, which will cause your detail.php action to be launched.

#### Parameters

When an action is selected, the remaining part of the URL is interpreted as a list of parameters.
In our case we have only one parameter: *pigs-ftw*. What will happen if the parameter(s) is valid or
not depends completely on how the action is developed.


### Backend

The backend routes are more structured then the frontend and are always in the same format:

```
http://myforksite.dev/private/<language>/<module>/<action>?<get-parameters>
```

No caching is applied here, accessing an backend URL will execute the provided module action.
