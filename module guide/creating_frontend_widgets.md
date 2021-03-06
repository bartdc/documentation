# Creating frontend-widgets

Widgets are items you can link to a block on a page. An example of a widget is a content-block. In this article we'll explain how to create your own widget that shows the last 5 tweets of your Twitter-account. We assume you have the basic-installation covered.

## Creating the widget

A widget can only live in a module, so if you don't have a module yet, you should create one. Creating a module isn't that hard:
First we will insert a new module in the modules-table, which can be done by executing the query below

```
INSERT INTO modules(name) VALUES ('Twitter');
```

Use the translations-module to add the translation for your module. In the example we will need to add an label with the reference code "Twitter". Once that is done, we should set the permissions

```
INSERT INTO groups_rights_modules(id, group_id, module) VALUES (NULL, '1', 'Twitter');
```

So, all database stuff is done, so fire up your favorite PHP-editor.

Open up the default_www/frontend/modules-folder and create a new folder with the same name as the module you inserted into the database, in our example we create a folder called twitter.

Each modules has the same structure, that's part of Fork, so we need to create some basic folder and files.

### The configuration-file

Each module must have a configuration-file, it defines some options. The file look the same across the modules, except for the class-name.

The class should live in the root of your module and should be called Config, in our example the class will be Frontend\Modules\Twitter\Config. You can copy/paste the code below.

```
 <?php

namespace Frontend\Modules\Twitter;

use Frontend\Core\Engine\Base\Config as FrontendBaseConfig;

/**
 * This is the configuration-object
 *
 * @package        frontend
 * @subpackage    twitter
 *
 * @author        Tijs Verkoyen <tijs@sumocoders.com>
 * @since        2.6
 */
final class Config extends FrontendBaseConfig
{
}
```

Because our widget doensn't have any configuration-stuff it is just an empty class, the functionality is extended from the base config-class.

### The code for the widget

Offcourse our widget will need some code to grab the tweets, or to authenticate if needed.

In a module we have different types of code: actions, ajax, model, widgets. For this howto we just need widgets, so we will create a folder called Widgets in the module.

The name I choose for the widget is *Stream*. I choose this name because in the future we can expand the module with new widgets, actions, ... So, the code for a widget is placed in a file with the same name as the widget. So create a file `Stream.php`.

Copy/paste the code below. The code is documented inline.

```
<?php

namespace Frontend\Modules\Twitter\Widgets;

use Frontend\Core\Engine\Base\Widget as FrontendBaseWidget;
use Frontend\Core\Engine\Model as FrontendModel;

/**
 * This is a widget with the twitter-stream
 * It will show do the oAuth-dance if no settings are stored. If the settings are stored it will grab the last
 *
 * @package        frontend
 * @subpackage    twitter
 *
 * @author        Tijs Verkoyen <tijs@sumocoders.com>
 * @author        Wouter Sioen <wouter@woutersioen.be>
 * @since        2.6
 */
class Stream extends FrontendBaseWidget
{
    /**
     * The number of tweets to show
     *
     * @var    int
     */
    const NUMBER_OF_TWEETS = 5;

    /**
     * An array that will hold all tweets we grabbed from Twitter.
     *
     * @var    array
     */
    private $tweets = array();

    /**
     * Execute the extra
     *
     * @return    void
     */
    public function execute()
    {
        // call parent
        parent::execute();

        // load template
        $this->loadTemplate();

        // parse
        $this->parse();
    }

    /**
     * Get the data from Twitter
     * This method is only executed if the template isn't cached
     *
     * @return    void
     */
    private function getData()
    {
        // for now we only support one account, in a later stadium we could implement multiple account.
        $id = 1;

        // init the application-variabled, alter these keys if you want to user your own application
        $consumerKey = 'K99h85EM3twFkqoRwvWRKw';
        $consumerSecret = 'tVQa9QYJiJifhWkhRZaGKSIaf1Keb4WvmmFloa6ClY';

        // grab the oAuth-tokens from the settings
        $oAuthToken = FrontendModel::getModuleSetting('Twitter', 'oauth_token_' . $id);
        $oAuthTokenSecret = FrontendModel::getModuleSetting('Twitter', 'oauth_token_secret_' . $id);

        // grab the user if from the settings
        $userId = FrontendModel::getModuleSetting('Twitter', 'user_id_' . $id);

        // require the Twitter-class
        require_once PATH_LIBRARY . '/external/twitter.php';

        // create instance
        $twitter = new \Twitter($consumerKey, $consumerSecret);

        // if the tokens aren't available we will start the oAuth-dance
        if($oAuthToken == '' || $oAuthTokenSecret == '')
        {
            // build url to the current page
            $url = SITE_URL . '/' . $this->URL->getQueryString();
            $chunks = explode('?', $url, 2);
            $url = $chunks[0];

            // check if we are in th second part of the authorization
            if(isset($_GET['oauth_token']) && isset($_GET['oauth_verifier']))
            {
                // get tokens
                $response = $twitter->oAuthAccessToken($_GET['oauth_token'], $_GET['oauth_verifier']);

                // store the tokens in the settings
                FrontendModel::setModuleSetting('Twitter', 'oauth_token_' . $id, $response['oauth_token']);
                FrontendModel::setModuleSetting('Twitter', 'oauth_token_secret_' . $id, $response['oauth_token_secret']);

                // store the user-id in the settings
                FrontendModel::setModuleSetting('Twitter', 'user_id_' . $id, $response['user_id']);

                // redirect to the current page, at this point the oAuth-dance is finished
                \SpoonHTTP::redirect($url);
            }

            // request a token
            $response = $twitter->oAuthRequestToken($url);

            // let the user authorize our widget
            $twitter->oAuthAuthorize($response['oauth_token']);
        }

        else
        {
            // set tokens
            $twitter->setOAuthToken($oAuthToken);
            $twitter->setOAuthTokenSecret($oAuthTokenSecret);

            // grab tweets
            $this->tweets = $twitter->statusesUserTimeline($userId, null);
        }
    }

    /**
     * Parse
     *
     * @return    void
     */
    private function parse()
    {
        // we will cache this widget for 10 minutes, so we don't have to call Twitter each time a visitor enters a page with this widget.
        $this->tpl->cache(FRONTEND_LANGUAGE . '_twitterWidgetStreamCache', (10 * 60 * 60));

        // if the widget isn't cached, we should grab the data from Twitter
        if(!$this->tpl->isCached(FRONTEND_LANGUAGE . '_twitterWidgetStreamCache'))
        {
            // get data from Twitter
            $this->getData();

            // init var
            $tweets = array();

            // build nice array which can be used in the template-engine
            foreach($this->tweets as $tweet)
            {
                // we don't want to show @replies, so skip tweets that have in_reply_to_user_id set.
                if($tweet['in_reply_to_user_id'] != '') continue;

                // init var
                $item = array();

                // add values we need
                $item['user_name'] = $tweet['user']['name'];
                $item['url'] = 'http://twitter.com/' . $tweet['user']['screen_name'] . '/status/' . $tweet['id'];
                $item['text'] = $tweet['text'];
                $item['created_at'] = strtotime($tweet['created_at']);

                // add the twee
                $tweets[] = $item;
            }

            // get the numbers
            $this->tpl->assign('widgetTwitterStream', array_slice($tweets, 0, self::NUMBER_OF_TWEETS));
        }
    }
}
```

As you read through the code you'll see that we use a Twitter-class. Download this class [here](https://github.com/tijsverkoyen/fork_frontend_twitter_widget/blob/master/twitter.php) and save it under /library/external/twitter.php. We use this class to communicate with Twitter.

The class names and folder structure in Fork are PSR compliant. This means that the class names and namespaces have the exact same structure as the folder structure. This is necessary for autoloading.

### The layout

Ok, the code is handled, now we need some layout. The layout folder contains the files that will be used to display what our code generates. Create a folder Layout/Widgets/ in the module.

And as for the code create a file with the same name as the widget, in our example: Stream.tpl
Copy/paste the code below. The code is documented inline.

```
{* We use the templates caching-feature as a way to limit the calls to the Twitter API *}
{cache:{$LANGUAGE}_twitterWidgetStreamCache}
    {* Only show the stream if there are tweets *}
    {option:widgetTwitterStream}
        <ul>
            {* Loop the tweets *}
            {iteration:widgetTwitterStream}
                <li>
                    {* By using the cleanupplaintext-modifier the links in the tweet will be parsed. *}
                    {$widgetTwitterStream.text|cleanupplaintext}

                    <p class="date">
                        <a href="{$widgetTwitterStream.url}">
                            <time datetime="{$widgetTwitterStream.created_at|date:'Y-m-d\TH:i:s'}" pubdate>
                                {$widgetTwitterStream.created_at|timeago}
                            </time>
                        </a>
                    </p>
                </li>
            {/iteration:widgetTwitterStream}
        </ul>
    {/option:widgetTwitterStream}
{/cache:{$LANGUAGE}_twitterWidgetStreamCache}
```

As you can see, we have the cache-tags in place with the same name (*{$LANGUAGE}*_twitterWidgetStreamCache) as we used in the code. This means the output will be cached. If there are tweets we will loop them en parse them in aan unordered list. als some meta-data is parsed.

## Inserting the extra

An extra is the item that will be linked to a block on a page. For the user to be able to choose our widget in the dropdownmenu we need to add it into the pages_extras.

```
INSERT INTO modules_extras(id, module, type, label, action, data, hidden, sequence) VALUES (NULL, 'Twitter', 'widget', 'Stream', 'Stream', NULL, 'N', '10000');
```

The action field needs to be the name of the widget, stream in our example.

## Linking your widget

Go to the pages-modules and edit the page whereon you would like the widget to appear on, select the template-tab and click the edit-icon for the block the widget should be linked to.

In the dialog, select widget as type, choose the module (Twitter) you created in the second dropdownmenu, in the third dropdownmenu you can select your newly created widget (Stream).

Once you saved the page, the widget is linked. If you open up the page you will be redirect to Twitter to authorize the application (the oAuth-dance), if you authorized the application you will see your tweets on the page.

That's all folks, enjoy your widgets!
