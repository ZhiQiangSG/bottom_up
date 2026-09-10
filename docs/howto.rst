How To - Project Documentation
======================================================================

Get Started
----------------------------------------------------------------------

Documentation can be written as rst files in `docs/`.

To build and serve docs with live reload, use the command::

    docker compose -f docker-compose.docs.yml up

Then open http://127.0.0.1:9000. The `docs` service (`compose/local/docs/Dockerfile`,
`command: /start-docs` → `make livehtml` in `docs/Makefile`) serves
`sphinx-autobuild` on port 9000.

Changes to files in `docs/`, `config/`, or `bottom_up/` will be picked up and reloaded automatically
(see the volume mounts in `docker-compose.docs.yml`). Build output goes to `docs/_build/`.

`Sphinx <https://www.sphinx-doc.org/>`_ is the tool used to build documentation.

Docstrings to Documentation
----------------------------------------------------------------------

The sphinx extension `apidoc <https://www.sphinx-doc.org/en/master/man/sphinx-apidoc.html>`_ is used to automatically document code using signatures and docstrings.

Numpy or Google style docstrings will be picked up from project files and available for documentation. See the `Napoleon <https://sphinxcontrib-napoleon.readthedocs.io/en/latest/>`_ extension for details.

For an in-use example, see the `page source <_sources/users.rst.txt>`_ for :ref:`users`.

To compile all docstrings automatically into documentation source files (`docs/api/`), run from the repo root:
    ::

        make -C docs apidocs

This can be done in the docs container:
    ::

        docker compose -f docker-compose.docs.yml run --rm docs make apidocs

(`docs/Makefile` runs `sphinx-apidoc -o ./api /app`; `/app` only exists inside the container.
Natively, point `APP` at the repo root, e.g. `make -C docs apidocs APP=..`.)
