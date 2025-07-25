This directory contains test cases for various Vim features.
The auxiliary functions to perform the tests are in the util/ folder.

For testing an indent script see runtime/indent/testdir/README.txt.
For testing a syntax script see runtime/syntax/testdir/README.txt.

If it makes sense, add a new test method to an already existing file.  You may
want to separate it from other tests with comment lines.

TO ADD A NEW STYLE TEST:

1) Create a test_<subject>.vim file.
2) Add test_<subject>.res to NEW_TESTS_RES in Make_all.mak in alphabetical
   order.
3) Also add an entry "test_<subject>" to NEW_TESTS in Make_all.mak.
4) Use make test_<subject> to run a single test.

At 2), instead of running the test separately, it can be included in
"test_alot".  Do this for quick tests without side effects.  The test runs a
bit faster, because Vim doesn't have to be started, one Vim instance runs many
tests.

At 4), to run a test in GUI, add "GUI_FLAG=-g" to the make command.


What you can use (see test_assert.vim for an example):

- Call assert_equal(), assert_true(), assert_false(), etc.

- Use assert_fails() to check for expected errors.

- Use try/catch to avoid an exception aborts the test.

- Use test_alloc_fail() to have memory allocation fail.  This makes it possible
  to check memory allocation failures are handled gracefully.  You need to
  change the source code to add an ID to the allocation.  Add a new one to
  alloc_id_T, before aid_last.

- Use test_override() to make Vim behave differently, e.g.  if char_avail()
  must return FALSE for a while.  E.g. to trigger the CursorMovedI autocommand
  event. See test_cursor_func.vim for an example.

- If the bug that is being tested isn't fixed yet, you can throw an exception
  with "Skipped" so that it's clear this still needs work.  E.g.: throw
  "Skipped: Bug with <c-e> and popupmenu not fixed yet"

- The following environment variables are recognized and can be set to
  influence the behavior of the test suite (see runtest.vim for details)

  - $TEST_MAY_FAIL=Test_channel_one    - ignore those failing tests
  - $TEST_FILTER=Test_channel    - only run test that match this pattern
  - $TEST_SKIP_PAT=Test_channel  - skip tests that match this pattern
  - $TEST_NO_RETRY=yes           - do not try to re-run failing tests
  You can also set them in Vim:
    :let $TEST_MAY_FAIL = 'Test_channel_one'
    :let $TEST_FILTER = '_set_mode'
    :let $TEST_SKIP_PAT = 'Test_loop_forever'
    :let $TEST_NO_RETRY = 'yes'
  Use an empty string to revert, e.g.:
    :let $TEST_FILTER = ''

- See the start of runtest.vim for more help.


TO ADD A SCREEN DUMP TEST:

Mostly the same as writing a new style test.  Additionally, see help on
"terminal-dumptest".  Put the reference dump in "dumps/Test_func_name.dump".


OLD STYLE TESTS:

There are a few tests that are used when Vim was built without the +eval
feature.  These cannot use the "assert" functions, therefore they consist of a
.in file that contains Normal mode commands between STARTTEST and ENDTEST.
They modify the file and the result gets written in the test.out file.  This
is then compared with the .ok file.  If they are equal the test passed.  If
they differ the test failed.


RUNNING THE TESTS:

To run a single test from the src directory:

    $ make test_<name>

The below commands should be run from the src/testdir directory.

To run a single test:

    $ make test_<name>.res

The file 'messages' contains the messages generated by the test script.  If a
test fails, then the test.log file contains the error messages.  If all the
tests are successful, then this file will be an empty file.

- To run a single test function from a test script:

    $ ../vim -u NONE -S runtest.vim <test_file>.vim <function_name>

- To execute only specific test functions, add a second argument:

    $ ../vim -u NONE -S runtest.vim test_channel.vim open_delay

- To run all the tests:

    $ make

- To run the test on MS-Windows using the MSVC nmake:

    > nmake -f Make_mvc.mak

- To run the tests with GUI Vim:

    $ make GUI_FLAG=-g

    or

    $ make VIMPROG=../gvim

- To cleanup the temporary files after running the tests:

    $ make clean


VIEWING GENERATED SCREENDUMPS (local):

You may also wish to look at the whole batch of failed screendumps after
running "make".  Source the "viewdumps.vim" script for this task:

    $ ../vim -u NONE -S viewdumps.vim \
				[dumps/Test_xxd_*.dump ...]

By default, all screendumps found in the "failed" directory will be added to
the argument list and then the first one will be loaded.  Loaded screendumps
that bear filenames of screendumps found in the "dumps" directory will be
rendering the contents of any such pair of files and the difference between
them (:help term_dumpdiff()); otherwise, they will be rendering own contents
(:help term_dumpload()).  Remember to execute :edit when occasionally you see
raw file contents instead of rendered.

At any time, you can add, list, and abandon other screendumps:

    :$argedit dumps/Test_spell_*.dump
    :args
    :qall

The listing of argument commands can be found under :help buffer-list.


VIEWING GENERATED SCREENDUMPS (from a CI-uploaded artifact):

After you have downloaded an artifact archive containing failed screendumps
and extracted its files in a temporary directory, you need to set up a "dumps"
directory by creating a symlink:

    $ cd /path/to/fork
    $ ln -s $(pwd)/src/testdir/dumps /tmp/src/testdir/dumps

You can now examine the extracted screendumps:

    $ ./src/vim -u NONE -S src/testdir/viewdumps.vim \
				/tmp/src/testdir/failed/*.dump


VIEWING GENERATED SCREENDUMPS (submitted for a pull request):

Note: There is also a "git difftool" extension described in ./commondumps.vim.

First, you need to check out the topic branch with the proposed changes and
write down a difference list between the HEAD commit (index) and its parent
commit with respect to the changed "dumps" filenames:

    $ cd /path/to/fork
    $ git switch prs/1234
    $ git diff-index --relative=src/testdir/dumps/ \
				--name-only prs/1234~1 > /tmp/filelist

Then, you need to check out the master branch, change the current working
directory to reconcile relative filepaths written in the filenames list,
possibly create an empty "failed" directory, copy in this directory the old
"dumps" files, whose names are on the same list, and follow it by checking out
the topic branch:

    $ git switch master
    $ cd src/testdir/dumps
    $ test -d ../failed || mkdir ../failed
    $ cp -t ../failed $(cat /tmp/filelist)
    $ git switch prs/1234

Make note of any missing new screendumps.  Please remember about the
introduced INVERTED relation between "dumps" and "failed", i.e. the files to
be committed are in "dumps" already and their old versions are in "failed".
Therefore, you need to copy the missing new screendumps from "dumps" to
"failed":

    $ cp -t ../failed foo_10.dump foo_11.dump foo_12.dump

After you have changed the current working directory to its parent directory,
you can now examine the screendumps from the "failed" directory (note that new
screendumps will be shown with no difference between their versions):

    $ cd ..
    $ ../vim -u NONE -S viewdumps.vim

