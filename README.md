# Ursus

Ursus is the static site generator used by [All About Berlin](https://allaboutberlin.com). It turns Markdown files and [Jinja](https://jinja.palletsprojects.com/) templates into a static website.

This project is in active use and development.

## Setup

### Installation

Install Ursus with pip:

```bash
pip install ursus-ssg
```

### Getting started

By default, Ursus looks for content in `./content`, and templates in `./templates`. It generates a website under `./output`.

Call `ursus` to build the project. Call `ursus --help` to see the command line options it supports.

Here is a simple piece of content. Save it under `./content/posts/first-post.md`.

```markdown
---
Title: Hello world!
Description: This is an example page
Date_created: 2022-10-10
---

## Hello beautiful world

*This* is a template. Pretty cool eh?
```

Here is a simple template. Save it under `./templates/posts/entry.html.jinja`. 

```
<!DOCTYPE html>
<html>
<head>
    <title>{{ entry.title }}</title>
    <meta name="description" content="{{ entry.description }}">
</head>
<body>
    {{ entry.body }}
</body>
</html>
```

Call `ursus` to generate this website. It will create `./output/posts/first-post.html`.

### Configuring Ursus

You can configure Ursus by creating a `ursus_config.py` file at the root of your project. When you call `ursus`, it will load this configuration.

```python
# Example Ursus config file
from ursus.config import config

config.content_path = Path(__file__).parent / 'blog'
config.templates_path = Path(__file__).parent / 'templates'
config.output_path = Path(__file__).parent.parent / 'dist'

config.site_url = 'https://allaboutberlin.com'

config.minify_js = True
config.minify_css = True
```

You can find all configuration options in `ursus/config.py`.

You can give your config a different name, and load it with the `-c` argument:

```bash
ursus -c path/to/config.py
```

## Basic concepts

### Content and Entries

**Content** is what fills your website: text, images, videos, PDFs. A single piece of content is called an **Entry**. The location of the Content is set by `config.content_path`. By default, it's under `./content`.

Content is usually *rendered* to create a working website. Some content (like Markdown files) is rendered with Templates, and other (like images) is converted to a different file format.

### Templates

**Templates** are used to render your Content. They are the theme of your website. The same templates can be applied to different Entries, or even reused for a different website. They are kept in a separate directory.

For example, a template can be the HTML that makes up the page around your content: the header, sidebar, and footer.

The location of the Templates is set by `config.templates_path`. By default, it's under `./templates`. You can have a different `templates_path` for each Generator.

For example:

- HTML templates that wrap a nice theme around your Content.
- Images and other static assets that are part of the website's theme

### Output

This is the final static website generated by Ursus.

The location of the Output is set by `config.output_path`. By default, it's under `./output`.

## How Ursus works

ContextProcessors transform the context, which is a dict with information about each of your Entries. Renderers use the context to know which pages to create, and what content to put in the templates.

In the example above, your context would look like this:

```
{
    'entries': {
        'posts/first-post.md': {
            'title': 'Hello world!',
            'description': 'This is an example page',
            'body': '<h2>Hello beautiful world</h2><p>...'
        }
    },
    'get_entries': function
    'globals': {},
}
```

### Generators

A **Generator** takes your Content and your Templates and produces an Output. It's a recipe to turn your content into a final result. The default **StaticSiteGenerator** generates a static website. You can write your own Generator to output an eBook, a PDF, or anything else.

#### StaticSiteGenerator

Generates a static website.

### Context processors

The context is a big object that is used to render templates.

A **ContextProcessor** fills this context object, or transforms its existing contents.

For example, the **MarkdownProcessor** generates the entry context out of a markdown file.

Only Entries with matching ContextProcessors are rendered. Entry or directory names that start with `.` or `_` are not rendered. You can use this to create drafts.

#### MarkdownProcessor

The `MarkdownProcessor` creates context for all `.md` files in `content_path`.

It makes a few changes to the default markdown output:

- Lazyload images (`loading=lazy`)
- Convert images to `<figure>` tags when appropriate
- Jinja tags (`{{ ... }}` and `{% ... %}`) are rendered as-is. You can use the, to `{% include %}` template parts and `{{ variables }}` in your content.
- Set the `srcset` to load responsive images from the `image_transforms` config.
- Put the front matter in the context
    - `Related_*` keys are replaced by a list of related entry dicts
    - `Date_` keys are converted to `datetime` objects

#### GetEntriesProcessor

The `GetEntriesProcessor` adds a `get_entries` method to the context. It's used to get a list of entries of a certain type, and sort it.

```jinja
{% set posts = get_entries('posts', filter_by=filter_function, sort_by='date_created', reverse=True) %}
```

### Renderers

**Renderer**s create content that make up the Output. In other words, they turn your content files into pages, correctly-sized images, RSS feeds, etc.

#### ImageTransformRenderer

Renders images in `content_path` with a few changes:

- Images are compressed and optimized.
- Images are resized according to the `image_transforms`. The images are shrunk if needed, but never stretched.
- Files that can't be transformed (PDF to PDF) are copied as-is to the output directory.
- Images that can't be resized (SVG to anything) are copied as-is to the output directory.
- Image EXIF data is removed.

This renderer does nothing unless `image_transforms` is set:
```python
config.image_transforms = {
    # ...
    'image_transforms': {
        # Default transform used as <img> src
        # Saved as ./output/path/to/image.jpg
        '': {
            'max_size': (3200, 4800),
        },
        # Saved as ./output/path/to/image.jpg and .webp
        'thumbnails': {
            'exclude': ('*.pdf', '*.svg'),  # glob patterns
            'max_size': (400, 400),
            'output_types': ('original', 'webp'),
        },
        # Only previews PDF files in specific locations
        # Saved as ./output/path/to/image.webp and .png
        'pdfPreviews': {
            'include': ('documents/*.pdf', 'forms/*.pdf'),  # glob patterns
            'max_size': (300, 500),
            'output_types': ('webp', 'png'),
        }
    },
    # ...
}
```

#### JinjaRenderer

Renders Content into Jinja templates using the context made by ContextProcessors.

A Template called `./output/hello-world.html.jinja` will be rendered as `./output/hello-world.html`. The template has access to anything you put in the context, including the `entries` dict, and the `get_entries` method.

A Template called `./output/posts/entry.html.jinja` will render all Entries under `./content/posts/*.md` and save them under `./output/posts/*.html`. The template has access to an `entry` variable.

Only Templates with the `.jinja` extension are rendered. Files or directory names that start with `.` or `_` are not rendered.

Files named `entry.*.jinja` are rendered once for each Entry with the same path. For example, `./templates/posts/entry.html.jinja` will render `./content/posts/hello-world.md`, `./content/posts/foo.md` and `./content/posts/bar.md`. The output path is the entry name with the extension replaced. If `./templates/posts/entry.html.jinja` renders `./templates/posts/hello-world.md`, the output file is `./output/posts/hello-world.html`.

All template files with the `.jinja` extension will be rendered. For example, `./templates/posts/index.html.jinja` will be rendered as `./output/posts/index.html`. Files starting with `_` are ignored.

The output path is the template name without the `.jinja` extension. For example, `index.html.jinja` will be rendered as `index.html`.

#### StaticAssetRenderer

Simply copies static assets (CSS, JS, images, etc.) under `./templates` to the same subdirectory in `./output`. Files starting with `.` are ignored. Files and directories starting with `_` are ignored.

It uses hard links instead of copying files. It's faster and it saves space.

## Getting started

1. **Create a directory** for your project. This is a sensible structure, because it works automatically with the default configuration:
    ```
    example_site/
    ├── ursus_config.py  # By default, Ursus will use this config file
    ├── templates/  # By default, Ursus will use this templates directory
    │   ├── index.html.jinja
    │   ├── css/
    │   │   └──style.css
    │   ├── js/
    │   │   └──scripts.js
    │   ├── fonts/
    │   │   ├── open-sans.svg
    │   │   ├── open-sans.ttf
    │   │   └── open-sans.woff
    │   └── posts/
    │       ├── index.html.jinja
    │       └── entry.html.jinja
    └── content/  # By default, Ursus will use this content directory
        ├── posts/
        │   ├── first-post.md
        │   ├── foo.md
        │   └── bar.md
        └── images/
            └── example.png
    ```
2. **Create a config file for your website.** You can copy `ursus/default_config.py`. If you call your config `ursus_config.py` and place it in your project root, it will be loaded automatically. Otherwise you must call ursus with the `-c` argument. If no config is set, Ursus will use the defaults set in `ursus/default_config.py`.
3. **Call the `ursus` command.**

#### Building from Sublime Text

You can configure Sublime Text to run Ursus when you press Cmd + B:

```json
// Sublime user settings or project config
{
    // ...
    "build_systems": [{
        "cmd": ["ursus", "-c", "$project_path/path/to/ursus_config.py"],
        "name": "Ursus",
    }],
    // ...
}

```