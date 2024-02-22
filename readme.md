# End User Computing developer portal
This repository has the files for the top level of the End User Computing (EUC)
developer portal proof-of-concept (PoC).

The top of the portal is an organisation (org) GitHub Pages website. Levels
below the top are repository GitHub Pages websites.

The name of the repository `EUCDigitalWorkspace.github.io` is a special name
recognised by GitHub. Top-level portal content is in the [docs/](docs/)
directory of this repository.

See the documentation here for notes about org and repository GitHub pages.  
[docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#types-of-github-pages-sites](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#types-of-github-pages-sites)

The org site also hosts these shared files and content.

-   Cascading Stylesheet (CSS) files that are or could be shared by more than
    one repository site, for example [docs/portal.css](docs/portal.css).
-   The favicon graphic, [docs/favicon.ico](docs/favicon.ico).

This repository also has these files.

-   The GIMP source of the favicon graphic [favicon.xcf](favicon.xcf).

-   A Python script, [testharness.py](testharness.py) that can be used to serve
    the site locally. Run the script like this for example.

        cd /wherever/you/cloned/EUCDigitalWorkspace.github.io
        python3 ./testharness.py
    
    Then open [localhost:8001](http://localhost:8001) in your browser. This
    facilitates work on the PoC. You can edit the files locally and see the
    results without having to push to GitHub.

-   A workspace declaration for use with VSCode or VSCodium,
    [here.code-workspace](here.code-workspace).

The script and workspace declaration assume you also clone the other
repositories that make up the portal as siblings, like this.

    /wherever/you/cloned/
    |
    +--- .github/
    |
    +--- EUCDigitalWorkspace.github.io/
    |
    +--- ws1-dev-centre/
    |
    +--- ws1-sdk-uem-android/

# Notes
All navigation links must have exactly one terminating slash /. Otherwise
relative paths for resources and navigation won't work.
