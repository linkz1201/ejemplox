# Chamilo 1.11.x

![PHP Composer](https://github.com/chamilo/chamilo-lms/workflows/PHP%20Composer/badge.svg?branch=1.11.x)
[![Scrut<?php
/* For licensing terms, see /license.txt */

use ChamiloSession as Session;

/**
 * @package chamilo.main
 */
define('CHAMILO_HOMEPAGE', true);
define('CHAMILO_LOAD_WYSIWYG', false);

/* Flag forcing the 'current course' reset, as we're not inside a course anymore. */
// Maybe we should change this into an api function? an example: CourseManager::unset();
$cidReset = true;
require_once 'main/inc/global.inc.php';

// The section (for the tabs).
$this_section = SECTION_CAMPUS; //rewritten below if including HTML file
$includeFile = !empty($_GET['include']);
if ($includeFile) {
    $this_section = SECTION_INCLUDE;
} elseif (api_get_configuration_value('plugin_redirection_enabled')) {
    RedirectionPlugin::redirectUser(api_get_user_id());
}

$header_title = null;
if (!api_is_anonymous()) {
    $header_title = ' ';
}

$controller = new IndexManager($header_title);
//Actions
$loginFailed = isset($_GET['loginFailed']) ? true : isset($loginFailed);
if (!empty($_GET['logout'])) {
    $redirect = !empty($_GET['no_redirect']) ? false : true;
    // pass $logoutInfo defined in local.inc.php
    $controller->logout($redirect, $logoutInfo);
}
/**
 * Registers in the track_e_default table (view in important activities in admin
 * interface) a possible attempted break in, sending auth data through get.
 *
 * @todo This piece of code should probably move to local.inc.php where the
 * actual login / logout procedure is handled.
 * The real use of this code block should be seriously considered as well.
 * This form should just use a security token and get done with it.
 */
if (isset($_GET['submitAuth']) && $_GET['submitAuth'] == 1) {
    $i = api_get_anonymous_id();
    Event::addEvent(
        LOG_ATTEMPTED_FORCED_LOGIN,
        'tried_hacking_get',
        $_SERVER['REMOTE_ADDR'].(empty($_POST['login']) ? '' : '/'.$_POST['login']),
        null,
        $i
    );
    echo 'Attempted breakin - sysadmins notified.';
    session_destroy();
    exit();
}
// Delete session item necessary to check for legal terms
if (api_get_setting('allow_terms_conditions') === 'true') {
    Session::erase('term_and_condition');
}
//If we are not logged in and customapages activated
if (!api_user_is_login() && CustomPages::enabled()) {
    if (Request::get('loggedout')) {
        CustomPages::display(CustomPages::LOGGED_OUT);
    } else {
        CustomPages::display(CustomPages::INDEX_UNLOGGED);
    }
}
/**
 * @todo This piece of code should probably move to local.inc.php where the
 * actual login procedure is handled.
 * @todo Check if this code is used. I think this code is never executed because
 * after clicking the submit button the code does the stuff
 * in local.inc.php and then redirects to index.php or user_portal.php depending
 * on api_get_setting('page_after_login').
 */
if (!empty($_POST['submitAuth'])) {
    // The user has been already authenticated, we are now to find the last login of the user.
    if (isset($_user['user_id'])) {
        $track_login_table = Database::get_main_table(TABLE_STATISTIC_TRACK_E_LOGIN);
        $sql = "SELECT UNIX_TIMESTAMP(login_date)
                FROM $track_login_table
                WHERE login_user_id = '".$_user['user_id']."'
                ORDER BY login_date DESC LIMIT 1";
        $result_last_login = Database::query($sql);
        if (!$result_last_login) {
            if (Database::num_rows($result_last_login) > 0) {
                $user_last_login_datetime = Database::fetch_array($result_last_login);
                $user_last_login_datetime = $user_last_login_datetime[0];
                Session::write('user_last_login_datetime', $user_last_login_datetime);
            }
        }
    }
} else {
    // Only if login form was not sent because if the form is sent the user was already on the page.
    Event::open();
}

if (!api_is_anonymous()) {
    $url = api_get_configuration_value('redirect_index_to_url_for_logged_users');
    if (!empty($url)) {
        header("Location: $url");
        exit;
    }
}

if (api_get_setting('display_categories_on_homepage') === 'true') {
    $controller->tpl->assign('course_category_block', $controller->return_courses_in_categories());
}
$controller->set_login_form();
//@todo move this inside the IndexManager

if (!api_is_anonymous()) {
    $controller->tpl->assign('profile_block', $controller->return_profile_block());
    $controller->tpl->assign('user_image_block', $controller->return_user_image_block());
    $controller->tpl->assign('course_block', $controller->return_course_block());
}
$hotCourses = '';
$announcements_block = '';

// Display the Site Use Cookie Warning Validation
$useCookieValidation = api_get_setting('cookie_warning');

if ($useCookieValidation === 'true') {
    if (isset($_POST['acceptCookies'])) {
        api_set_site_use_cookie_warning_cookie();
    } elseif (!api_site_use_cookie_warning_cookie_exist()) {
        if (Template::isToolBarDisplayedForUser()) {
            $controller->tpl->assign('toolBarDisplayed', true);
        } else {
            $controller->tpl->assign('toolBarDisplayed', false);
        }
        $controller->tpl->assign('displayCookieUsageWarning', true);
    }
}
// When loading a chamilo page do not include the hot courses and news
if (!isset($_REQUEST['include'])) {
    if (api_get_setting('show_hot_courses') == 'true') {
        if (api_get_configuration_value('popular_courses_handpicked')) {
            // If the option has been set correctly, use the courses manually
            // marked as popular rather than the ones marked by users
            $hotCourses = $controller->returnPopularCoursesHandPicked();
        } else {
            $hotCourses = $controller->return_hot_courses();
        }
    }
    $announcements_block = $controller->return_announcements();
}
if (api_get_configuration_value('show_hot_sessions') === true) {
    $hotSessions = SessionManager::getHotSessions();
    $controller->tpl->assign('hot_sessions', $hotSessions);
}
$controller->tpl->assign('hot_courses', $hotCourses);
$controller->tpl->assign('announcements_block', $announcements_block);

$allowJustification = api_get_plugin_setting('justification', 'tool_enable') === 'true';

$justification = '';
if ($allowJustification) {
    $plugin = Justification::create();
    $courseId = api_get_plugin_setting('justification', 'default_course_id');
    if (!empty($courseId)) {
        $courseInfo = api_get_course_info_by_id($courseId);
        $link = Display::url($plugin->get_lang('SubscribeToASession'), $courseInfo['course_public_url']);
        $justification = Display::return_message($link, 'info', false);
    }
}

if ($includeFile) {
    // If we are including a static page, then home_welcome is empty
    $controller->tpl->assign('home_welcome', $justification);
    $controller->tpl->assign('home_include', $controller->return_home_page($includeFile));
} else {
    // If we are including the real homepage, then home_include is empty
    $controller->tpl->assign('home_welcome', $justification.$controller->return_home_page(false));
    $controller->tpl->assign('home_include', '');
}
$controller->tpl->assign('navigation_links', $controller->return_navigation_links());
$controller->tpl->assign('notice_block', $controller->return_notice());
$controller->tpl->assign('help_block', $controller->return_help());
$controller->tpl->assign('student_publication_block', $controller->studentPublicationBlock());
if (api_is_platform_admin() || api_is_drh()) {
    $controller->tpl->assign('skills_block', $controller->returnSkillLinks());
}
if (api_is_anonymous()) {
    $controller->tpl->setLoginBodyClass();
}
// direct login to course
if (isset($_GET['firstpage'])) {
    $firstPage = $_GET['firstpage'];
    $courseInfo = api_get_course_info($firstPage);

    if (!empty($courseInfo)) {
        api_set_firstpage_parameter($firstPage);

        // if we are already logged, go directly to course
        if (api_user_is_login()) {
            echo "<script>self.location.href='index.php?firstpage=".Security::remove_XSS($firstPage)."'</script>";
        }
    }
} else {
    api_delete_firstpage_parameter();
}

$controller->setGradeBookDependencyBar(api_get_user_id());
$controller->tpl->display_two_col_template();

// Deleting the session_id.
Session::erase('session_id');
Session::erase('id_session');
Session::erase('studentview');
api_remove_in_gradebook();
inizer Code Quality](https://scrutinizer-ci.com/g/chamilo/chamilo-lms/badges/quality-score.png?b=1.11.x)](https://scrutinizer-ci.com/g/chamilo/chamilo-lms/?branch=1.<?php
/* For licensing terms, see /license.txt */

use ChamiloSession as Session;

/**
 * @package chamilo.main
 */
define('CHAMILO_HOMEPAGE', true);
define('CHAMILO_LOAD_WYSIWYG', false);

/* Flag forcing the 'current course' reset, as we're not inside a course anymore. */
// Maybe we should change this into an api function? an example: CourseManager::unset();
$cidReset = true;
require_once 'main/inc/global.inc.php';

// The section (for the tabs).
$this_section = SECTION_CAMPUS; //rewritten below if including HTML file
$includeFile = !empty($_GET['include']);
if ($includeFile) {
    $this_section = SECTION_INCLUDE;
} elseif (api_get_configuration_value('plugin_redirection_enabled')) {
    RedirectionPlugin::redirectUser(api_get_user_id());
}

$header_title = null;
if (!api_is_anonymous()) {
    $header_title = ' ';
}

$controller = new IndexManager($header_title);
//Actions
$loginFailed = isset($_GET['loginFailed']) ? true : isset($loginFailed);
if (!empty($_GET['logout'])) {
    $redirect = !empty($_GET['no_redirect']) ? false : true;
    // pass $logoutInfo defined in local.inc.php
    $controller->logout($redirect, $logoutInfo);
}
/**
 * Registers in the track_e_default table (view in important activities in admin
 * interface) a possible attempted break in, sending auth data through get.
 *
 * @todo This piece of code should probably move to local.inc.php where the
 * actual login / logout procedure is handled.
 * The real use of this code block should be seriously considered as well.
 * This form should just use a security token and get done with it.
 */
if (isset($_GET['submitAuth']) && $_GET['submitAuth'] == 1) {
    $i = api_get_anonymous_id();
    Event::addEvent(
        LOG_ATTEMPTED_FORCED_LOGIN,
        'tried_hacking_get',
        $_SERVER['REMOTE_ADDR'].(empty($_POST['login']) ? '' : '/'.$_POST['login']),
        null,
        $i
    );
    echo 'Attempted breakin - sysadmins notified.';
    session_destroy();
    exit();
}
// Delete session item necessary to check for legal terms
if (api_get_setting('allow_terms_conditions') === 'true') {
    Session::erase('term_and_condition');
}
//If we are not logged in and customapages activated
if (!api_user_is_login() && CustomPages::enabled()) {
    if (Request::get('loggedout')) {
        CustomPages::display(CustomPages::LOGGED_OUT);
    } else {
        CustomPages::display(CustomPages::INDEX_UNLOGGED);
    }
}
/**
 * @todo This piece of code should probably move to local.inc.php where the
 * actual login procedure is handled.
 * @todo Check if this code is used. I think this code is never executed because
 * after clicking the submit button the code does the stuff
 * in local.inc.php and then redirects to index.php or user_portal.php depending
 * on api_get_setting('page_after_login').
 */
if (!empty($_POST['submitAuth'])) {
    // The user has been already authenticated, we are now to find the last login of the user.
    if (isset($_user['user_id'])) {
        $track_login_table = Database::get_main_table(TABLE_STATISTIC_TRACK_E_LOGIN);
        $sql = "SELECT UNIX_TIMESTAMP(login_date)
                FROM $track_login_table
                WHERE login_user_id = '".$_user['user_id']."'
                ORDER BY login_date DESC LIMIT 1";
        $result_last_login = Database::query($sql);
        if (!$result_last_login) {
            if (Database::num_rows($result_last_login) > 0) {
                $user_last_login_datetime = Database::fetch_array($result_last_login);
                $user_last_login_datetime = $user_last_login_datetime[0];
                Session::write('user_last_login_datetime', $user_last_login_datetime);
            }
        }
    }
} else {
    // Only if login form was not sent because if the form is sent the user was already on the page.
    Event::open();
}

if (!api_is_anonymous()) {
    $url = api_get_configuration_value('redirect_index_to_url_for_logged_users');
    if (!empty($url)) {
        header("Location: $url");
        exit;
    }
}

if (api_get_setting('display_categories_on_homepage') === 'true') {
    $controller->tpl->assign('course_category_block', $controller->return_courses_in_categories());
}
$controller->set_login_form();
//@todo move this inside the IndexManager

if (!api_is_anonymous()) {
    $controller->tpl->assign('profile_block', $controller->return_profile_block());
    $controller->tpl->assign('user_image_block', $controller->return_user_image_block());
    $controller->tpl->assign('course_block', $controller->return_course_block());
}
$hotCourses = '';
$announcements_block = '';

// Display the Site Use Cookie Warning Validation
$useCookieValidation = api_get_setting('cookie_warning');

if ($useCookieValidation === 'true') {
    if (isset($_POST['acceptCookies'])) {
        api_set_site_use_cookie_warning_cookie();
    } elseif (!api_site_use_cookie_warning_cookie_exist()) {
        if (Template::isToolBarDisplayedForUser()) {
            $controller->tpl->assign('toolBarDisplayed', true);
        } else {
            $controller->tpl->assign('toolBarDisplayed', false);
        }
        $controller->tpl->assign('displayCookieUsageWarning', true);
    }
}
// When loading a chamilo page do not include the hot courses and news
if (!isset($_REQUEST['include'])) {
    if (api_get_setting('show_hot_courses') == 'true') {
        if (api_get_configuration_value('popular_courses_handpicked')) {
            // If the option has been set correctly, use the courses manually
            // marked as popular rather than the ones marked by users
            $hotCourses = $controller->returnPopularCoursesHandPicked();
        } else {
            $hotCourses = $controller->return_hot_courses();
        }
    }
    $announcements_block = $controller->return_announcements();
}
if (api_get_configuration_value('show_hot_sessions') === true) {
    $hotSessions = SessionManager::getHotSessions();
    $controller->tpl->assign('hot_sessions', $hotSessions);
}
$controller->tpl->assign('hot_courses', $hotCourses);
$controller->tpl->assign('announcements_block', $announcements_block);

$allowJustification = api_get_plugin_setting('justification', 'tool_enable') === 'true';

$justification = '';
if ($allowJustification) {
    $plugin = Justification::create();
    $courseId = api_get_plugin_setting('justification', 'default_course_id');
    if (!empty($courseId)) {
        $courseInfo = api_get_course_info_by_id($courseId);
        $link = Display::url($plugin->get_lang('SubscribeToASession'), $courseInfo['course_public_url']);
        $justification = Display::return_message($link, 'info', false);
    }
}

if ($includeFile) {
    // If we are including a static page, then home_welcome is empty
    $controller->tpl->assign('home_welcome', $justification);
    $controller->tpl->assign('home_include', $controller->return_home_page($includeFile));
} else {
    // If we are including the real homepage, then home_include is empty
    $controller->tpl->assign('home_welcome', $justification.$controller->return_home_page(false));
    $controller->tpl->assign('home_include', '');
}
$controller->tpl->assign('navigation_links', $controller->return_navigation_links());
$controller->tpl->assign('notice_block', $controller->return_notice());
$controller->tpl->assign('help_block', $controller->return_help());
$controller->tpl->assign('student_publication_block', $controller->studentPublicationBlock());
if (api_is_platform_admin() || api_is_drh()) {
    $controller->tpl->assign('skills_block', $controller->returnSkillLinks());
}
if (api_is_anonymous()) {
    $controller->tpl->setLoginBodyClass();
}
// direct login to course
if (isset($_GET['firstpage'])) {
    $firstPage = $_GET['firstpage'];
    $courseInfo = api_get_course_info($firstPage);

    if (!empty($courseInfo)) {
        api_set_firstpage_parameter($firstPage);

        // if we are already logged, go directly to course
        if (api_user_is_login()) {
            echo "<script>self.location.href='index.php?firstpage=".Security::remove_XSS($firstPage)."'</script>";
        }
    }
} else {
    api_delete_firstpage_parameter();
}

$controller->setGradeBookDependencyBar(api_get_user_id());
$controller->tpl->display_two_col_template();

// Deleting the session_id.
Session::erase('session_id');
Session::erase('id_session');
Session::erase('studentview');
api_remove_in_gradebook();
.x)
[![Bountysource](https://www.bountysource.com/badge/team?team_id=12439&style=raised)](https://www.bountysource.com/teams/chamilo?utm_source=chamilo&utm_medium=shield&utm_campaign=raised)
[![Code Consistency](https://squizlabs.github.io/PHP_CodeSniffer/analysis/chamilo/chamilo-lms/grade.svg)](http://squizlabs.github.io/PHP_CodeSniffer/analysis/chamilo/chamilo-lms/)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/166/badge)](https://bestpractices.coreinfrastructure.org/projects/166)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/88e934aab2f34bb7a0397a6f62b078b2)](https://www.codacy.com/app/chamilo/chamilo-lms?utm_source=github.com&utm_medium=referral&utm_content=chamilo/chamilo-lms&utm_campaign=badger)

## Installation

This installation guide is for development environments only.

### Install PHP, a web server and MySQL/MariaDB

To run Chamilo, you will need at least a web server (we recommend Apache2 for commodity reasons), a database server (we recommend MariaDB but will explain MySQL for commodity reasons) and a PHP interpreter (and a series of libraries for it). If you are working on a Debian-based system (Debian, Ubuntu, Mint, etc), just
type
```
sudo apt-get install apache2 mysql-server php libapache2-mod-php php-gd php-intl php-curl php-json php-mysql php-zip composer
```

### Install Git

The development version 1.11.x requires you to have Git installed. If you are working on a Debian-based system (Debian, Ubuntu, Mint, etc), just type
```
sudo apt-get install git
```

### Install Composer

To run the development version 1.11.x, you need Composer, a libraries dependency management system that will update all the libraries you need for Chamilo to the latest available version.

Make sure you have Composer installed. If you do, you should be able to launch "composer" on the command line and have the inline help of composer show a few subcommands. If you don't, please follow the installation guide at https://getcomposer.org/download/

### Download Chamilo from GitHub

Clone the repository

```
sudo mkdir chamilo-1.11
sudo chown -R `whoami` chamilo-1.11
git clone -b 1.11.x --single-branch https://github.com/chamilo/chamilo-lms.git chamilo-1.11
```

Checkout branch 1.11.x

```
cd chamilo-1.11
git checkout --track origin/1.11.x
git config --global push.default current
```

### Update dependencies using Composer

From the Chamilo folder (in which you should be now if you followed the previous steps), launch:

```
composer update
```

If you face issues related to missing JS libraries, you might need to ensure
that your web/assets folder is completely re-generated.
Use this set of commands to do that:
```
rm composer.lock
rm -rf web/ vendor/
composer clear-cache
composer update
```
This will take several minutes in the best case scenario, but should definitely
generate the missing files.

### Change permissions

On a Debian-based system, launch:
```
sudo chown -R www-data:www-data app main/default_course_document/images main/lang web
```

### Configure the web server

Enable the Apache web server module "rewrite" :
```
sudo a2enmod rewrite
sudo systemctl restart apache2.service
```

Chamilo's .htaccess must be obeyed.
Create /etc/apache2/conf-available/htaccessForChamilo.conf with these lines :
```
<Directory /var/www/html/chamilo-lms>
	AllowOverride All
</Directory>
```

then enable it :
```
sudo a2enconf htaccessForChamilo
sudo systemctl reload apache2.service
```

If you just installed missing PHP extensions using apt, you must restart the web server to get them loaded :
```
sudo systemctl restart apache2.service
```

### Start the installer

In your browser, load the Chamilo URL. You should be automatically redirected 
to the installer. If not, add the "main/install/index.php" suffix manually in 
your browser address bar. The rest should be a matter of simple
 OK > Next > OK > Next...

## Upgrade from 1.10.x

1.11.0 is a major version. It contains a series of new features, that
also mean a series of new database changes in regards with versions 1.10.x. As 
such, it is necessary to go through an upgrade procedure when upgrading from 
1.10.x to 1.11.x.

The upgrade procedure is relatively straightforward. If you have a 1.10.x 
initially installed with Git, here are the steps you should follow 
(considering you are already inside the Chamilo folder):
```
git fetch --all
git checkout origin 1.11.x
```

Then load the Chamilo URL in your browser, adding "main/install/index.php" and 
follow the upgrade instructions. Select the "Upgrade from 1.10.x" button to 
proceed.

If you have previously updated database rows manually, you might face issue with
FOREIGN KEYS during the upgrade process. Please make sure your database is
consistent before upgrading. This usually means making sure that you have to delete
rows from tables referring to rows which have been deleted from the user or access_url tables.
Typically:
<pre>
    DELETE FROM access_url_rel_course WHERE access_url_id NOT IN (SELECT id FROM access_url);
</pre>

### Upgrading from non-Git Chamilo 1.10 ###

In the *very unlikely* case of upgrading a "normal" Chamilo 1.10 installation (done with the downloadable zip package) to a Git-based installation, make sure you delete the contents of a few folders first. These folders are re-generated later by the ```composer update``` command. This is likely to increase the downtime of your Chamilo portal of a few additional minutes (plan for 10 minutes on a reasonnable internet connection).

```
rm composer.lock
rm -rf web/*
rm -rf vendor/*
```


# For developers and testers only

This section is for developers only (or for people who have a good reason to use
a development version of Chamilo), in the sense that other people will not 
need to update their Chamilo portal as described here.

## Updating code

To update your code with the latest developments in the 1.11.x branch, go to
your Chamilo folder and type:
```
git pull origin 1.11.x
```
If you have made customizations to your code before the update, you will have
two options:
- abandon your changes (use "git stash" to do that)
- commit your changes locally and merge (use "git commit" and then "git pull")

You are supposed to have a reasonable understanding of Git in order to
use Chamilo as a developer, so if you feel lost, please check the Git manual
first: http://git-scm.com/documentation

## Updating your database from new code

Since the 2015-05-27, Chamilo offers the possibility to make partial database
upgrades through Doctrine migrations.

To update your database to the latest version, go to your Chamilo root folder
and type
```
php bin/doctrine.php migrations:migrate --configuration=app/config/migrations.yml
```

If you want to proceed with a single migration "step" (the steps reside in
src/Chamilo/CoreBundle/Migrations/Schema/V110/), then check the datetime of the
version and type the following (assuming you want to execute Version20150527120703)
```
php bin/doctrine.php migrations:execute 20150527120703 --up --configuration=app/config/migrations.yml
```

You can also print the differences between your database and what it should be by issuing the following command from the Chamilo base folder:
```
php bin/doctrine.php orm:schema-tool:update --dump-sql
```

## Contributing

If you want to submit new features or patches to Chamilo, please follow the
Github contribution guide https://guides.github.com/activities/contributing-to-open-source/
and our CONTRIBUTING.md file.
In short, we ask you to send us Pull Requests based on a branch that you create
with this purpose into your repository forked from the original Chamilo repository.

# Documentation
For more information on Chamilo, visit https://1.11.chamilo.org/documentation/index.html
