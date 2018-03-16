## Buildspec.yaml Files

This folder has a bunch of buildspec files.

These files are used by AWS CodeBuild to determine how to build your project. Included are buildspec.yaml files for:

1. Identity transformation - This copies the entire Git
   repository content to the static website with no
   modifications.

2. Subdirectory  - This is useful if your
   Git repository has files that should not be included as part of the
   static site. It publishes a specified subdirectory (e.g., "htdocs"
   or "public-html") as the static website, keeping the rest of your
   repository private.

3. Hugo - This runs the popular
   [Hugo][hugo] static site generator. The Git repository should
   include all source templates, content, theme, and config.

4. Jekyll plugin - This runs the popular
   [Jekyll][jekyll] static site generator. The Git repository should
   include all source templates, content, theme, and config.

[jekyll]: https://jekyllrb.com/
[hugo]: https://gohugo.io/


The relevant buildspec.yml should be copied into the root of your project directory, and can be adapted with any custom commands you need to build your project.

Feel free to create your own buildspec.yml files and contribute them back if you feel they might be useful to others!