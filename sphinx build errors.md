[[Vicharak]]

sphinxawesome-theme
breathe
myst_parser
sphinx_design
sphinx_favicon
sphinx_togglebutton
sphinxcontrib.spelling
sphinx-tabs
linkify-it-py


make clean
rm -rf _build

file source/_static
Also check _templates similarly.

the problem - Sphinx expects _templates to be a directory (even if empty) because it's referenced in your conf.py:

templates_path = ['_templates']


the fix - mkdir -p source/_templates



prev - 5f5a0bca77e5f0894d0b29ee9b3835df7d59edc4
my - 652e79c19c7a0b62c2b2f94d5e97f4390783db21

/home/ronin/Desktop/Vicharak/Vaaman/vicharak-docs/_build

