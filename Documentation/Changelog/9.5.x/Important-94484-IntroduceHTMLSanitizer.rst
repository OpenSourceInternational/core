.. include:: ../../Includes.txt

============================================
Important: #94484 - Introduce HTML Sanitizer
============================================

See :issue:`94484`

Description
===========

To sanitize and purge XSS from markup during frontend rendering, new
custom HTML sanitizer has been introduced, based on `masterminds/html5`.
Both :php:`\TYPO3\HtmlSanitizer\Builder\CommonBuilder` and
:php:`\TYPO3\HtmlSanitizer\Visitor\CommonVisitor` provide common configuration
which is in line with expected tags that are allowed in backend RTE.

Using a custom builder instance, it is possible to adjust for individual
demands - however, configuration possibilities cannot be modified using
TypoScript - basically since the existing syntax does not cover all
necessary scenarios.

PHP API
=======

The API is considered "internal", however it might be necessary to provide
custom markup handling, add additional tags, attributes or values. The whole
process of sanitization is based on an "allow-list" - everything that is not
allowed, is automatically denied.

The following example is meant to give a brief overview of the behavior and
corresponding possibilities.

.. code-block:: php

   <?php
   use TYPO3\CMS\Core\Html\DefaultSanitizerBuilder;
   use TYPO3\HtmlSanitizer\Behavior;
   use TYPO3\HtmlSanitizer\Builder\BuilderInterface;

   class MyCustomBuilder extends DefaultSanitizerBuilder implements BuilderInterface
   {
       public function createBehavior(): Behavior
       {
           // extends existing behavior, adds new tag
           return parent::createBehavior()
               ->withName('my-custom')
               ->withTags(
                   (new Behavior\Tag('my-element', Behavior\Tag::ALLOW_CHILDREN))
                   ->addAttrs(
                       (new Behavior\Attr('href'))->addValues(
                           new Behavior\RegExpAttrValue('#^(?:https?://|mailto:)#')
                       ),
                       ...$this->globalAttrs
                   )
               );
           }
       }
   }

As a result a new tag :html:`my-element` is which is allowed to
* have any safe global attribute (`id`, `class`, `data-*`, ...)
* have attribute `href`, in case corresponding value either starting with `http://`,
  `http://` or `mailto:` - evaluated from the given regular expression


TypoScript
==========

stdWrap.htmlSanitize
--------------------

New :typoscript:`stdWrap` property :typoscript:`htmlSanitize` has been introduced
to control sanitization of markup, removing tags, attributes or values that have
not been allowed explicitly.

* `htmlSanitize = [boolean]` whether to invoke sanitization (enabled per default).
* `htmlSanitize.build = [string]` defines which specific builder (must be an
  instance of :php:`\TYPO3\HtmlSanitizer\Builder\BuilderInterface`)
  to be used for building a :php:`\TYPO3\HtmlSanitizer\Sanitizer`
  instance using a particular :php:`\TYPO3\HtmlSanitizer\Behavior`.
  This can either be a fully qualified class name or the name of a preset as
  defined in :php:`$GLOBALS['TYPO3_CONF_VARS']['SYS']['htmlSanitizer']` - per
  default, :php:`\TYPO3\CMS\Core\Html\DefaultSanitizerBuilder` is used.

.. code-block:: typoscript

   10 = TEXT
   10 {
     value = <div><img src="invalid.file" onerror="alert(1)"></div>
     htmlSanitize = 1
     // htmlSanitize.build = default
     // htmlSanitize.build = TYPO3\CMS\Core\Html\DefaultSanitizerBuilder
   }


stdWrap.parseFunc
-----------------

:typoscript:`stdWrap.htmlSanitize` is enabled per default when
:typoscript:`stdWrap.parseFunc` is invoked. This also includes Fluid
view-helper :html:`<f:format.html>`, since it invokes :php:`parseFunc`
using :typoscript:`lib.parseFunc_RTE` directly.

The following example shows how sanitization behavior - enabled per default -
can be disabled. This is not recommended, but occasionally might be necessary.

.. code-block:: typoscript

   // either disable globally
   lib.parseFunc.htmlSanitize = 0
   lib.parseFunc_RTE.htmlSanitize = 0

   // or disable individually per use-case
   10 = TEXT
   10 {
     value = <div><img src="invalid.file" onerror="alert(1)"></div>
     parseFunc =< lib.parseFunc_RTE
     parseFunc.htmlSanitize = 0
   }


Backend RTE configuration
=========================

Processing instructions for rich-text fields in the backend user interface
can be adjusted in a similar way, e.g. in :file:`Configuration/Processing.yaml`.

.. code-block:: yaml

   processing:
     allowTags:
       # ...
     HTMLparser_db:
       # ...
       htmlSanitize:
         # use default builder as configured in
         # $GLOBALS['TYPO3_CONF_VARS']['SYS']['htmlSanitizer']
         build: default

       # disable individually per use-case
       # htmlSanitize: false

Sanitization for persisting data can be disabled globally using corresponding
feature flag :php:`$GLOBALS['TYPO3_CONF_VARS']['SYS']['features']['rte.htmlSanitize']`.


.. index:: Backend, Frontend, ext:core
