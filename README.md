# drewmalin.github.io

**Notes to Self**

### Structure

**Main page** is driven from: [_data/menu.yaml](_data/menu.yaml).

**Posts** are placed in: [_posts](_posts)

**Archive pages** are (generally) automatically created using the 'category' metadata within a given post.

**HTML pages** within the [_includes](_includes) or [_layout](_layout) directories will override the template's default counterparts for these pages. The following have been created to provide new functionality for this site:
* [_includes/head.html](_includes/head.html) - modified to include custom CSS that allows for code syntax highlighting in posts
* [_layout/post.html](_layout/post.html) - modified to include the list of categories in the header of the post
* [_layout/project.html](_layout/project.html) - created to support displaying all posts that are related to a specific project (driven by the 'which_tag' metadata tag on a given post)


### Run locally

```bash
> bundle exec jekyll serve
```

