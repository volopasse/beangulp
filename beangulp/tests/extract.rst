Setup
-----

  >>> import pathlib
  >>> from os import mkdir, path, unlink
  >>> from shutil import rmtree
  >>> from tempfile import mkdtemp
  >>> import click.testing
  >>> import beangulp
  >>> from beangulp.tests.utils import Importer, IdentityImporter

An importer to create error conditions:

  >>> class ErrorImporter(Importer):
  ...
  ...     def extract(self, filepath, existing):
  ...         name = path.basename(filepath)
  ...         raise ValueError(name)

Test harness:

  >>> importers = [
  ...     Importer('test.ImporterA', 'Assets:Tests', 'text/csv'),
  ...     ErrorImporter('test.ImporterE', 'Assets:Tests', None),
  ...     IdentityImporter('test.Identity1', 'Assets:Tests', '*one.beans'),
  ...     IdentityImporter('test.Identity2', 'Assets:Tests', '*two.beans'),
  ... ]
  >>> runner = click.testing.CliRunner()
  >>> def run(*args):
  ...     ingest = beangulp.Ingest(importers)
  ...     return runner.invoke(ingest.cli, args, catch_exceptions=False)

Create a dowloads directory:

  >>> temp = mkdtemp()
  >>> downloads = path.join(temp, 'downloads')
  >>> mkdir(downloads)


Tests
-----

The basics:

  >>> r = run('extract', '--help')
  >>> r.exit_code
  0
  >>> print(r.output)
  Usage: beangulp extract [OPTIONS] [SRC]...

Test with an empty downloads directory:

  >>> r = run('extract', downloads)
  >>> r.exit_code
  0
  >>> print(r.output)

Add some documents:

  >>> fnames = ['aaa.txt', 'bbb.csv', 'zzz.txt']
  >>> for fname in fnames:
  ...     with open(path.join(downloads, fname), 'w') as f:
  ...         pass

  >>> output = path.join(temp, 'output.beancount')
  >>> r = run('extract', downloads, '-o', output)
  >>> r.exit_code
  0
  >>> print(r.output)
  * .../downloads/aaa.txt
  * .../downloads/bbb.csv ... OK
  * .../downloads/zzz.txt

Check the output file:

  >>> with open(output, 'r') as f:
  ...     extracted = f.read()
  >>> print(extracted)
  ;; -*- mode: beancount -*-
  <BLANKLINE>
  **** .../downloads/bbb.csv
  <BLANKLINE>

Test importer raising an error:

  >>> with open(path.join(downloads, 'error.foo'), 'w') as f:
  ...     pass

  >>> output = path.join(temp, 'output.beancount')
  >>> r = run('extract', downloads, '-o', output)
  >>> r.exit_code
  1
  >>> print(r.output)
  * .../downloads/aaa.txt
  * .../downloads/bbb.csv ... OK
  * .../downloads/error.foo ... ERROR
    Exception in importer code.
    Traceback (most recent call last):
      ...
    ValueError: error.foo
  * .../downloads/zzz.txt

Check the output file:

  >>> with open(output, 'r') as f:
  ...     extracted = f.read()
  >>> print(extracted)
  ;; -*- mode: beancount -*-
  <BLANKLINE>
  **** .../downloads/bbb.csv
  <BLANKLINE>

Cleanup:

  >>> rmtree(downloads)
  >>> mkdir(downloads)


Deduplication
-------------

Test the identity importer:

  >>> existing = path.join(temp, 'existing.beancount')
  >>> _ = pathlib.Path(downloads).joinpath('one.beans').write_text("""
  ... 2023-01-01 * "Test"
  ...   Assets:Tests  2 TESTS
  ... """)
  >>> r = run('extract', downloads, '-o', existing)
  >>> r.exit_code
  0
  >>> print(r.output)
  * .../downloads/one.beans ... OK

  >>> print(pathlib.Path(existing).read_text())
  ;; -*- mode: beancount -*-
  <BLANKLINE>
  **** .../downloads/one.beans
  <BLANKLINE>
  2023-01-01 * "Test"
    Assets:Tests  2 TESTS

Importing again the same file results in entries marked as duplicates:

  >>> r = run('extract', downloads, '-o', output, '-e', existing)
  >>> r.exit_code
  0
  >>> print(r.output)
  * .../downloads/one.beans ... OK

  >>> print(pathlib.Path(output).read_text())
  ;; -*- mode: beancount -*-
  <BLANKLINE>
  **** .../downloads/one.beans
  <BLANKLINE>
  ; duplicate of .../existing.beancount:5
  ; 2023-01-01 * "Test"
  ;   Assets:Tests  2 TESTS

  >>> _ = pathlib.Path(downloads).joinpath('two.beans').write_text("""
  ... 2023-01-01 * "Test"
  ...   Assets:Tests  2 TESTS
  ... """)
  >>> r = run('extract', downloads, '-o', output)
  >>> r.exit_code
  0
  >>> print(pathlib.Path(output).read_text())
  ;; -*- mode: beancount -*-
  <BLANKLINE>
  **** .../downloads/one.beans
  <BLANKLINE>
  2023-01-01 * "Test"
    Assets:Tests  2 TESTS
  <BLANKLINE>
  **** .../downloads/two.beans
  <BLANKLINE>
  ; .../downloads/one.beans:2
  ; 2023-01-01 * "Test"
  ;   Assets:Tests  2 TESTS


Per-account output directory
----------------------------

The ``--output-dir`` (``-d``) option writes one file per account, named
after the account with ``:`` replaced by ``.``, creating the directory if
it does not exist:

  >>> outdir = path.join(temp, 'by-account')
  >>> r = run('extract', downloads, '-d', outdir)
  >>> r.exit_code
  0
  >>> outfile = pathlib.Path(outdir, 'Assets.Tests.beancount')
  >>> outfile.read_text().splitlines()[0]
  ';; -*- mode: beancount -*-'
  >>> [l for l in outfile.read_text().splitlines() if l.startswith('****')]
  ['**** .../downloads/one.beans', '**** .../downloads/two.beans']

Re-running appends without repeating the mode header:

  >>> r = run('extract', downloads, '-d', outdir)
  >>> r.exit_code
  0
  >>> outfile.read_text().count(';; -*- mode: beancount -*-')
  1
  >>> len([l for l in outfile.read_text().splitlines() if l.startswith('****')])
  4

A missing leading dot on ``--ext`` is added:

  >>> r = run('extract', downloads, '-d', outdir, '--ext', 'bc')
  >>> r.exit_code
  0
  >>> path.exists(path.join(outdir, 'Assets.Tests.bc'))
  True

Batches with no account go to ``unfiled.beancount``:

  >>> _ = pathlib.Path(downloads).joinpath('unfiled.beans').write_text("""
  ... 2023-01-01 * "Test"
  ...   Assets:Tests  4 TESTS
  ... """)
  >>> noaccount = beangulp.Ingest([
  ...     IdentityImporter('test.NoAccount', None, '*unfiled.beans')])
  >>> r = runner.invoke(noaccount.cli, ['extract', downloads, '-d', outdir],
  ...                   catch_exceptions=False)
  >>> r.exit_code
  0
  >>> path.exists(path.join(outdir, 'unfiled.beancount'))
  True

Combining ``-d`` with an explicit ``--output`` is rejected before any
file is created:

  >>> conflict = path.join(temp, 'conflict.beancount')
  >>> r = run('extract', downloads, '-d', outdir, '-o', conflict)
  >>> r.exit_code
  2
  >>> path.exists(conflict)
  False

Extraction to standard output is unchanged:

  >>> r = run('extract', downloads)
  >>> r.exit_code
  0
  >>> ';; -*- mode: beancount -*-' in r.output
  True


Cleanup
-------

  >>> rmtree(temp)
