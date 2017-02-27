# AWS CloudFormation Docs

This repository hosts documentation for the AWS CloudFormation deployment methodology used by Casecommons.

See https://casecommons.github.io/aws-docs for the full documentation.

## Modifying the Docs

To run and modify the docs locally, you must have [Hugo installed](https://gohugo.io/#action).

After you have installed Hugo, clone the repository locally, ensuring you specify the `--recursive` flag to clone the [Material Docs theme](https://github.com/digitalcraftsman/hugo-material-docs/):

```
$ git clone --recursive git@github.com:Casecommons/aws-docs.git
```

### Running the Docs Locally

To run the docs locally:

```
$ hugo server
Started building sites ...
Built site for language en:
0 draft content
0 future content
0 expired content
9 regular pages created
9 other pages created
0 non-page files copied
0 paginator pages created
0 tags created
0 categories created
total in 30 ms
Watching for changes in /Users/jmenga/Source/casecommons/aws-docs/{content,layouts,static,themes}
Serving pages from memory
Web Server is available at http://localhost:1313/aws-docs/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

You will be able to access the docs at http://localhost:1313/aws-docs/.  Any changes you make to the local repository will be hot reloaded.

See the [Hugo documentation](https://gohugo.io/overview/introduction/) for details on how to modify content.


