beangulp: Importers Framework for Beancount
-------------------------------------------

``beangulp`` provides a framework for importing transactions into a
Beancount ledger from account statements and other documents and for
managing documents.

``beangulp`` is compatible with both Beancount 2.3 and the
Beancount 3.0. ``beangulp`` is the evolution of ``beancount.ingest`` in
Beancount 2.3 and replaces it in the Beancount 3.0 release.

Documentation is available in form of code docstrings and
`examples`__.  The `documentation`__ for ``beancount.ingest`` provided
a good introduction and is still partially relevant to ``beangulp``.
The rest of the Beancount documentation can be found `here`__.

__ https://github.com/beancount/beangulp/tree/master/examples/
__ https://beancount.github.io/docs/importing_external_data/
__ https://beancount.github.io/docs/


Extracting to per-account files
-------------------------------

The ``extract`` command writes the extracted entries to the standard
output by default, or to a single file with ``--output``/``-o``. Pass
``--output-dir``/``-d`` to instead write one flat Beancount file per
account, named after the account with ``:`` replaced by ``.`` (for
example ``Assets.Denis.Chase.Current.beancount``). The directory is
created if it does not exist, and ``--ext`` selects the file extension
(default ``.beancount``; a missing leading dot is added). Batches that are
not associated with an account are written to ``unfiled.beancount``.

Existing files are appended to, so re-running the extraction adds the
new sections without removing earlier content and without repeating the
``;; -*- mode: beancount -*-`` header. Appending does not de-duplicate
across runs; use ``--existing``/``-e`` for de-duplication. The ``-d``
option cannot be combined with an explicit ``-o``.
