# blog

To build the website locally, run:

1. `bundle install`
2. `bundle exec jekyll serve`
3. Visit `http://localhost:4000/blog/`

_Based on the instructions here: <https://github.com/orgs/community/discussions/37669#discussioncomment-4079487>._

Reading times are calculated automatically at build time by `_includes/reading-time.html`, using 200 words per minute, rounded up to at least one minute. The estimate counts the rendered post's text (including code) after stripping HTML; it does not add time for images. The home and post layouts override Minima 2.5.1 only to display the estimate beside the date. No additional plugins or JavaScript are required.
