This folder contains the source used to generate a static site using Nikola.

Installation and documentation at https://getnikola.com/:
```bash
python3 -m venv pyenv
source pyenv/bin/activate
pip3 install -r requirements.txt
nikola theme -i bootstrap3-jinja  # my theme is based on this one
```

Configuration file for the site is `conf.py`.

To build the site::

    nikola build

To see it::

    nikola serve -b

To check all available commands::

    nikola help