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

The code highlighting features including in the documentation require the Python library `pygments` to be installed locally and the ability to call the command `pygmentize` from a local shell:

Assuming you have the Python package manager `pip` installed, to install `pygments`:

```
$ pip install pygments
Requirement already satisfied: pygments in /usr/local/lib/python2.7/site-packages
$ pygmentize --help
Usage: /usr/local/bin/pygmentize [-l <lexer> | -g] [-F <filter>[:<options>]] [-f <formatter>]
          [-O <options>] [-P <option=value>] [-s] [-v] [-x] [-o <outfile>] [<infile>]
```

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

## License

Copyright (C) 2017.  Case Commons, Inc.

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU Affero General Public License as published by the Free
Software Foundation, either version 3 of the License, or (at your option) any
later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

See www.gnu.org/licenses/agpl.html

## Release Notes

### Version 0.2.0

- **NEW FEATURE** : Added misc section covering the use of generic certificates template `https://github.com/Casecommons/aws-docs/pull/6`

### Version 0.1.0

- **NEW FEATURE** : Added sections covering dns configuration and orchestration of the frontend app `https://github.com/Casecommons/aws-docs/pull/5`

### Version 0.0.1

- **BUG FIX** : Fixed errors present in current documentation `https://github.com/Casecommons/aws-docs/pull/4`


