---
draft: true
title: Contribution Guidelines
description: How to contribute to LEIKAH
weight: 10
---

We use [Hugo](https://gohugo.io/) to format and generate our website and the
[Docsy](https://github.com/docsy/docsy) theme for styling and site structure.
Hugo is an open-source static site generator that provides us with templates,
content organisation in a standard directory structure, and a website generation
engine. You write the pages in Markdown (or HTML if you want), and Hugo wraps them up into a website.

All submissions, including submissions by project members, require review. We
use GitHub pull requests for this purpose. Consult
[GitHub Help](https://help.github.com/articles/about-pull-requests/) for more
information on using pull requests.

## Updating a single page

If you've just spotted something you'd like to change while using the docs, Docsy has a shortcut for you:

1. Click **Edit this page** in the top-right corner of the page.
1. If you don't already have an up-to-date fork of the project repo, you are prompted to get one - click **Fork this repository and propose changes** or **Update your Fork** to get an up-to-date version of the project to edit. The appropriate page in your fork is displayed in edit mode.
1. Follow the rest of the [Previewing your changes locally](#previewing-your-changes-locally) process below to make, preview, and propose your changes.

## Previewing your changes locally

If you want to run your own local Hugo server to preview your changes as you work:

1. Install **Node.js 26** and **Go 1.26.3 or later**. The project installs its pinned Hugo 0.170.0 and Dart Sass 1.106.1 toolchain through npm.
1. Fork the [LEIKAH repo](https://github.com/RandomChugokujin/leikah) into your own project, then create a local copy using `git clone`.

    ```
    git clone https://github.com/RandomChugokujin/leikah.git
    ```

1. Run `npm ci`, then `npm run serve` in the site root directory. By default your site will be available at http://localhost:1313/. Now that you're serving your site locally, Hugo will watch for changes to the content and automatically refresh your site.
1. Continue with the usual GitHub workflow to edit files, commit them, push the
  changes up to your fork, and create a pull request.

## Creating an issue

If you've found a problem in the docs, but you're not sure how to fix it yourself, please create an issue in the [LEIKAH repo](https://github.com/RandomChugokujin/leikah/issues). You can also create an issue about a specific page by clicking the **Create Issue** button in the top-right corner of the page.

## Useful resources

* [Docsy user guide](https://www.docsy.dev/docs/): All about Docsy, including how it manages navigation, look and feel, and multi-language support.
* [Hugo documentation](https://gohugo.io/documentation/): Comprehensive reference for Hugo.
* [GitHub Hello World!](https://guides.github.com/activities/hello-world/): A basic introduction to GitHub concepts and workflow.
