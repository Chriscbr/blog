# blog

## Development

Try the instructions here: <https://github.com/orgs/community/discussions/37669#discussioncomment-4079487>

With some changes like:

```diff
diff --git a/Gemfile b/Gemfile
new file mode 100644
index 0000000..75d9835
--- /dev/null
+++ b/Gemfile
@@ -0,0 +1,2 @@
+source "https://rubygems.org"
+gem "github-pages", group: :jekyll_plugins
diff --git a/_config.yml b/_config.yml
index 54d66e4..44df972 100644
--- a/_config.yml
+++ b/_config.yml
@@ -1,5 +1,6 @@
 title: "functional fascinations"
 author: "Chris Rybicki"
+baseurl: /blog

 theme: minima
 ```
