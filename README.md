rhdhorchestrator.github.io

This site is built by hugo static site generater and published using a github action to https://rhdhorchestrator.io

# Devleopment
- Requirements
    - go
    - git
    - Download `hugo` *extended* version from [hugo releases page](https://github.com/gohugoio/hugo/releases/)       or run
      ```bash
      sudo snap install hugo
      ```

- Run it
    ```bash
    hugo server
    ```
    If you encounter cache issue, ie: remote Markdown file not updated in Hugo, you can disable it by adding the `--ignoreCache` flag:
    ```bash
    hugo server --ignoreCache
    ```

# Content Organization

- content/docs \
  The main directory for the project document
- content/docs/workflows \
  Document for the selected set of workflows, for https://www.rhdhorchestrator.io/main/docs/serverless-workflows
- content/docs/workflows-examples \
  Document for the examples workflows, for https://www.rhdhorchestrator.io/main/docs/serverless-workflows/examples
- content/post \
  Articles, blog posts, etc.

Read more the on hugo documentation https://gohugo.io/documentation/

# How to add a document?
Documents can include markdown content from all the related *`rhdhorchestrator`* repositories.
To create a document entry from a markdown file use this:

```bash
./generate-doc-for-repo.sh \
    https://github.com/rhdhorchestrator/$REPO/blob/$BRANCH//README.md > content/docs/newdoc.md
```
