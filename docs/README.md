Gluu Enterprise Edition (EE) Documentation
======================================

This is the home of the documentation for Gluu Server Enterprise Edition, which is 
built using mkdocs, and is rendered as html on the [Gluu Documentation site]
(https://www.gluu.org/docs-enterprise-edition/)
.

# Mkdocs Installation / Site Generation

Install virtualenv if necessary:

```
$ sudo pip install virtualenv
```

Create a local python enviroment and install the version of mkdocs we used to make our custom theme:

```
$ virtualenv env
$ env/bin/pip install -r requirements.txt
```

For more information see their website: http://www.mkdocs.org

To generate the documentation:

     $ env/bin/mkdocs build

This will create a directory called `site` which has the `index.html`
for viewing in your browser.

To run the server locally:

     $ env/bin/mkdocs serve

The home repository for this project is:

- [https://github.com/GluuFederation/docs-cluster](https://github.com/GluuFederation/docs-cluster)

If you would like to contribute documentation please post a message on
 http://support.gluu.org


> License
>
> Copyright © 2015 Gluu, Inc.
>
> Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
