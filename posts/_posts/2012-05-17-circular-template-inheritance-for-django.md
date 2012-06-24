---
layout: post
title: Circular Template Inheritance for Django
tags:
- django
- python
- mezzanine
- open source
---

One of Django's many great features is its powerful [template inheritance][template-inheritance]. A decade ago we would use simple concepts like include files for platforms such as ASP and PHP, allowing us to create reusable template snippets that could be embedded in multiple pages. Later, ASP.NET and Ruby on Rails would improve on this with their master/layout concepts, allowing us to define a base skeleton template for a site, with an area that all other pages would inject their content into. Django takes this approach even further with its template inheritance. It allows templates to extend other (parent) templates, with those parent templates containing named blocks that can be overridden. The blocks in the parent template can contain default content, and when overriding these blocks, the default content can be overridden, left as is, or even prefixed with or appended to, as the child template will have access to the default content in the parent template's block. This is analogous to object oriented programming, where base classes can be subclassed, and have their methods overridden, with access to the super-class's methods to be called at whatever point is deemed appropriate.

#### Overriding vs Extending

Another powerful feature of Django's is its [template loaders][template-loaders]. Each loader implements an approach for finding and loading the contents of a template when it's requested by name. A typical Django project will contain multiple template loaders, and when a template is loaded by name, that name will be passed through each of the loaders sequentially, until one of the loaders finds the template.

The two most commonly used template loaders are the ``filesystem`` loader and the ``app_directories`` loader. The ``filesystem`` loader will look at the ``TEMPLATE_DIRS`` setting, and search each of the directory paths in it for the requested template. The ``app_directories`` loader is similar, but will look for a directory called "templates" in each of the apps listed in the ``INSTALLED_APPS`` setting.

{% highlight python %}
TEMPLATE_LOADERS = (
    "django.template.loaders.filesystem.Loader",
    "django.template.loaders.app_directories.Loader",
)

PROJECT_ROOT = os.path.dirname(os.path.abspath(__file__))
TEMPLATE_DIRS = (os.path.join(PROJECT_ROOT, "templates"),)
{% endhighlight %}

Reusable Django apps will often provide a default set of templates where applicable, and the app's view functions will load these templates by name. With both the ``filesystem`` and ``app_directories`` template loaders configured, the app's version of the template will be loaded, unless a template with the same name is found in the project's templates directory, since the ``filesystem`` loader is listed first in the ``TEMPLATE_LOADERS`` setting. This allows a project developer to easily override the templates for a third-party app by copying it into their project's templates directory, in order to customise the look and feel of the app.

A problem arises for the project developer however, when the app's template contains sufficiently complex features, like many extendable template blocks, template variables, and more. Once they copy the template to their project's templates directory, they're essentially forking it, that is, they'll no longer be able to seamlessly make use of any new features for that template in future versions of the third-party app. Worst case is that an upgrade to the app will break their project, until they copy the new version of the template and customize it again, or upgrade their own copy by hand to be compatible with the latest version of the app.

With a complex template like this, more often than not the project developer may simply want to change it in a very small way, such as modifying the content in one of its blocks. Wouldn't it be nice if you could use template inheritance to extend the app's template, and simply override the relevant blocks as desired? Unfortunately this isn't possible with Django due to circular template inheritance. The app's view will be looking for the template to load by a given name. If we want our project's version to be used, we need to use the same template name for it to be loaded. If our project's version of the template tries to extend a template with the same name, Django will load our project's template again when looking for the parent template to extend, resulting in an infinite loop that will never complete. Django's template inheritance isn't smart (or stupid) enough to ignore the absolute path of the current template being used, when searching for the parent template to extend.

#### Alternative Approaches

The general approach to dealing with this problem, is for app developers to separate the name of of the template being loaded by their view, from the parts of the template that a project developer may want to customise. This might involved breaking all of the features up into separate include files that can be overridden individually. Another approach is to make each view load an empty template that extends the real template - developers can then override the empty template, end
extend the real template as required.

In an effort to make [Mezzanine][mezzanine]'s templates more customisable, these ideas were recently [brainstormed on the Mezzanine mailing list][mailing-list-template-thread]. While these approaches might work extremely well for individual Django apps that only provide a handful of default templates, the question of complexity and maintenance comes up with larger-scale projects like Mezzanine which contains almost 100 template files at the moment. All of a sudden we're looking at a minimum of doubling the number of template files - even more if we get more granular with includes. We've lost the simplicity of simply checking which template a view loads, and copying it to our project for customisation. So the question was proposed as to how could we possibly get circular template inheritance to work - if that was possible, we'd have a fantastic tool for both overriding and extending templates at once, without any wide-sweeping changes to the template structure across the entire project. Read on for the gory details of how it's quite possible.

#### Hacks Inside

Fair warning: the rest of this post describes an unorthodox approach that allows circular template inheritance to work. It's a little bit crazy, it's a little bit cool. Some would call it a terrible hack, your mileage may vary. If you're going to use it, consider the actual problem it solves, and whether or not it applies to your situation. Understand what it does, test it, and weigh it up against the alternative approaches described earlier.

The problem can be boiled down to one of state - when Django's ``extends`` template tag is used and the parent template to extend is searched for, the tag contains no knowledge of the extending template calling it. If it did, we could theoretically exclude its path from those being used to search for the parent template. So a possible solution might involve two steps:

* Make the ``extends`` tag aware of the path for the template that's calling it.
* Search for a template with the same name, preferably using Django's template loaders, and exclude the path of the template doing the extending.

Both of these steps are achievable thanks to the design of Django's template loaders. While the underlying location of a template isn't available through the higher level interfaces like ``get_template`` and ``find_template``, if you dig down slightly into the template API, you'll find each loader contains a ``load_template_source`` method, which is part of the call chain for finding and loading a template's source. ``load_template_source`` returns both the source for the template it discovers, as well as the absolute path of the template.

From this point we could go ahead and fulfill the second step by searching all other possible file-system paths for a template with the same relative template path, excluding the absolute path we've retrieved via ``load_template_source``, but ideally we'd like to leverage the template loaders to do this. Fortunately this is a breeze given the way template loaders work. The ``load_template_source`` method for each of the template loaders can accept a list of directories to use, and will fall back to a default if none are specified. The ``filesystem`` loader will use the directories defined by the ``TEMPLATE_DIRS`` setting, and the ``app_directories`` loader will use a list of template directories for all of the ``INSTALLED_APPS`` which it builds up when first loaded. I've never seen these directory arguments used in practice, but there they are just begging to be exploited as the perfect solution to our template searching problem.

#### Extending Extends

Now that we have the theoretical hooks needed, it's time to implement our template tag. Again we find ourselves in the situation where the pieces of Django we need to touch are structured perfectly to do what we need. The ``extends`` tag is implemented using the ``ExtendsNode`` class. It contains a ``get_parent`` method, which is responsible for loading the parent template object that is being extended. So all we need to do is subclass ``ExtendsNode`` and override ``get_parent``. We'll also include our own ``find_template`` method, similar to Django's, that also returns the absolute path of the template that was found. I've dubbed this approach "overextending", since it allows you to both override and extend a template at the same time.

{% highlight python %}
from django.conf import settings
from django.template import TemplateDoesNotExist
from django.template.loader_tags import ExtendsNode

class OverExtendsNode(ExtendsNode):

    def find_template(self, name, dirs):
        """
        Replacement for Django's ``find_template``, that also returns
        the location of the template found.
        """
        for loader_name in settings.TEMPLATE_LOADERS:
            loader = find_template_loader(loader_name)
            try:
                source, path = loader.load_template_source(name, dirs)
            except TemplateDoesNotExist:
                pass
            else:
                return source, path
        raise TemplateDoesNotExist(name)

    def get_parent(self, context):
        """
        Load the parent template using our own ``find_template``, which
        will also give us the path of the template found. We then peek
        at the first node, and if its parent arg is the same as the
        current parent arg, we know circular inheritance is going to
        occur, in which case we try and find the template again, with
        the absolute directory removed from the search list.
        """
        from django.template.loaders.app_directories import app_template_dirs
        dirs = list(settings.TEMPLATE_DIRS + app_template_dirs)
        parent = self.parent_name.resolve(context)
        template, path = self.find_template(parent, dirs)
        if (isinstance(template.nodelist[0], ExtendsNode) and
            template.nodelist[0].parent_name.resolve(context) == parent):
            # Remove the template directory from the available
            # directories to search in, and try again.
            dirs.remove(path[:-len(parent) - 1])
            template, path = self.find_template(parent, dirs)
        return template
{% endhighlight %}

All that remains is creating the ``overextends`` template tag function that uses ``OverExtendsNode``. For this we can pretty much copy pasta Django's ``extends`` tag function, replacing ``ExtendsNode`` with ``OverExtendsNode``.

#### Diving Deeper - Unlimited Inheritance Levels

Keeping in mind that this approach is is specifically geared towards solving the problem of both overriding and extending a template in a third party app, that is, one we don't want to modify the source code of, our approach so far works. But what if we wanted to overextend a template that also overextends _another_ template? Say for example our project template overextends a template in third-party app "A", which is dependent on a template in third-party "B", that it _also_ overextends. This is quite an edge case, but the code above would fail in this scenario. When the template in app "A" tries to overextend the template in app "B", it would exclude itself from the search path, and end up loading our project's version of the template, and we're back to square one with circular inheritance never completing.

This is less of an edge case and much more likely, with frameworks like Mezzanine, where it's common for people to create themes as third-party apps. The theme provides a set of templates and static files, and gets added to ``INSTALLED_APPS`` just as a regular Django app would. With the approach of overextending introduced into the eco-system, theme developers may overextend Mezzanine's templates, and project developers may overextend the theme's templates. Taking this into account, our approach so far looks quite handicapped.

Again we have a state problem - the second and subsequent calls to ``overextends`` have no knowledge of the previous calls, so they can't exclude the chain of template directories that have been so far excluded when overextending.

We can solve this problem by making use of the template context to store the state of directories excluded so far when using ``overextends``. We store a dictionary mapping template names to lists of directories available. In the code above, we build the full list of directories to use each time ``overextends`` is called. If we maintain that list in the template context, removing from it each time ``overextends`` is used, we can support unlimited levels of circular template inheritance.

Here's an refactored version of the previous example, along with the ``overextends`` tag function, supporting multiple levels of circular template inheritance.

{% highlight python %}
from django import template
from django.template import Template, TemplateSyntaxError, TemplateDoesNotExist
from django.template.loader_tags import ExtendsNode
from django.template.loader import find_template_loader

register = template.Library()

class OverExtendsNode(ExtendsNode):

   def find_template(self, name, context):
        """
        Replacement for Django's ``find_template`` that uses the current
        template context to keep track of which template directories it
        has used when finding a template. This allows multiple templates
        with the same relative name/path to be discovered, so that
        circular template inheritance can occur.
        """

        # These imports want settings, which aren't available when this
        # module is imported to ``add_to_builtins``, so do them here.
        from django.template.loaders.app_directories import app_template_dirs
        from django.conf import settings

        # Store a dictionary in the template context mapping template
        # names to the lists of template directories available to
        # search for that template. Each time a template is loaded, its
        # origin directory is removed from its directories list.
        context_name = "OVEREXTENDS_DIRS"
        if context_name not in context:
            context[context_name] = {}
        if name not in context[context_name]:
            all_dirs = list(settings.TEMPLATE_DIRS + app_template_dirs)
            context[context_name][name] = all_dirs

        # Build a list of template loaders to use. For loaders that wrap
        # other loaders like the ``cached`` template loader, unwind its
        # internal loaders and add those instead.
        loaders = []
        for loader_name in settings.TEMPLATE_LOADERS:
            loader = find_template_loader(loader_name)
            loaders.extend(getattr(loader, "loaders", [loader]))

        # Go through the loaders and try to find the template. When
        # found, removed its absolute path from the context dict so
        # that it won't be used again when the same relative name/path
        # is requested.
        for loader in loaders:
            dirs = context[context_name][name]
            try:
                source, path = loader.load_template_source(name, dirs)
            except TemplateDoesNotExist:
                pass
            else:
                context[context_name][name].remove(path[:-len(name) - 1])
                return Template(source)
        raise TemplateDoesNotExist(name)

    def get_parent(self, context):
        """
        Load the parent template using our own ``find_template``, which
        will cause its absolute path to not be used again. Then peek at
        the first node, and if its parent arg is the same as the
        current parent arg, we know circular inheritance is going to
        occur, in which case we try and find the template again, with
        the absolute directory removed from the search list.
        """
        parent = self.parent_name.resolve(context)
        # If parent is a template object, just return it.
        if hasattr(parent, "render"):
            return parent
        template = self.find_template(parent, context)
        if (isinstance(template.nodelist[0], ExtendsNode) and
            template.nodelist[0].parent_name.resolve(context) == parent):
            return self.find_template(parent, context)
        return template

@register.tag
def overextends(parser, token):
    """
    Extended version of Django's ``extends`` tag that allows circular
    inheritance to occur, eg a template can both be overridden and
    extended at once.
    """
    bits = token.split_contents()
    if len(bits) != 2:
        raise TemplateSyntaxError("'%s' takes one argument" % bits[0])
    parent_name = parser.compile_filter(bits[1])
    nodelist = parser.parse()
    if nodelist.get_nodes_by_type(ExtendsNode):
        raise TemplateSyntaxError("'%s' cannot appear more than once "
                                  "in the same template" % bits[0])
    return OverExtendsNode(nodelist, parent_name, None)
{% endhighlight %}

The final step required is to automatically add our ``overextends`` tag to Django's built-in tags. Django's ``ExtendsNode`` uses a feature where it gets marked as having to be the first tag in a template (``ExtendsNode.must_be_first`` is set to ``True``). This means that it (and subsequently our ``ExtendsNode`` subclass) need to be available without having to load the template library that implements it. This is as simple as calling the ``django.template.loader.add_to_builtins`` function from your project's settings module, passing it the Python dotted path as a string for the module that contains out ``overextends`` tag.

#### django-overextends

Originally this post contained a similar approach to the one above, but made use of the ``origin`` attribute found on template objects. Shortly after publishing it, [Tobia Conforto][tobia-conforto] helped me work out that the ``origin`` attributes on template objects are only available when ``DEBUG`` is ``True`` in your Django project. A big thanks goes out to him for bringing this up, allowing me to work out a more solid approach that this post now describes.

Tobia also suggested how badly a solution like this is needed for reusable app developers, and that it really belongs in a publicly available package that people can install and add to their Django projects. So I've bundled it into a reusable Django application called [django-overextends][django-overextends]. It contains documentation, tests, [continuous integration][travis-ci], and is available on [PyPI], [GitHub] and [Bitbucket].

[template-inheritance]: https://docs.djangoproject.com/en/dev/topics/templates/#template-inheritance
[template-loaders]: https://docs.djangoproject.com/en/dev/ref/templates/api/#loader-types
[mezzanine]: http://mezzanine.jupo.org
[mailing-list-template-thread]: https://groups.google.com/group/mezzanine-users/browse_thread/thread/b14c9d71ffc86644
[tobia-conforto]: https://github.com/tobia
[django-overextends]: http://github.com/stephenmcd/django-overextends
[travis-ci]: http://travis-ci.org/#!/stephenmcd/django-overextends
[pypi]: http://pypi.python.org/pypi/django-overextends
[github]: http://github.com/stephenmcd/django-overextends
[bitbucket]: http://bitbucket.org/stephenmcd/django-overextends

