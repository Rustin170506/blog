rustin.me
============

Blog posts, presentations, logs, and other things...

> fork from [jakewharton.com](https://github.com/JakeWharton/jakewharton.com)

Development
===========

Ensure you have the correct version of Ruby installed (per `.ruby-version`). Also ensure you have the Bundler gem
installed.

```sh
# Install rbenv
brew install rbenv

# Initialize rbenv
rbenv init

# Add to ~/.zshrc or ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.zshrc

# Install Ruby version from .ruby-version
rbenv install $(cat .ruby-version)

# Install bundler
gem install bundler:2.4.10
```

One-time setup
--------------

```sh
bundle install
```

_Note: If you're on Mac OS and this fails installing nokogiri, run `brew unlink xz`, install, and then `brew link xz`._

Running the site
----------------

```sh
bundle exec jekyll serve
```

License
=======

The content of the site (blog posts, presentation slides, etc.) are licensed
as [Creative Commons CC BY 4.0](https://creativecommons.org/licenses/by/4.0/legalcode).

The code powering the site is licensed as:

    Copyright 2017 Jake Wharton
    Copyright 2019 Rustin

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
